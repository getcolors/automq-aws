# Live AWS verification — 2026-09-10

AutoMQ deployed successfully under profile `automq-aws`, using colors-compute
for all compute. A second complete converge using published pins also passed.

## Configuration and provenance

- AWS account `251213589273`, region `us-east-1`, availability zone `us-east-1a`.
- Three Ubuntu 24.04 `t3.large` broker/controllers; three encrypted 40 GiB root volumes.
- AutoMQ `1.7.4`, image digest `sha256:68bf5df674ab9755da51f5200c152df391b0968aeeaf9ec4d12619517cd1234f`.
- Replication factor 1; durability uses S3 data storage.
- Lifecycle-created S3 backend, data and ops buckets, with a scoped application IAM identity.
- Private CA with IP SANs; clients verify the CA and IP identity. No DNS provider required.
- [AutoMQ source](https://github.com/getcolors/automq/commit/e3beeaf08472fc3ab4e5121eca876e1f0bb6f8e9)
  and [launcher publication](https://github.com/getcolors/automq/commit/debb50b).
- [colors-compute runtime](https://github.com/getcolors/colors-compute/commit/87ec5661fc8807159419d90d437247f3503469c7).

The first successful run used local development overrides. The second fetched
the published AutoMQ commit through the copied Green launcher, with
`AUTOMQ_LIB_ROOT`, `COLORS_COMPUTE_LIB_ROOT`, `GREEN_LIB_ROOT`, and `ONCE_LIB_ROOT`
unset. Blue and Red received automated validation; they were not exercised live.

## Results

| Check | Observed result |
|---|---|
| Initial live create and complete acceptance | Passed after fixing the issues below |
| Published-pin repeat converge | Passed |
| On-host gates | Three registered brokers and quorum voters; exact 500-record round trip; objects in both app buckets; authentication and ACL refusals |
| External gates | 16 passed, 0 failed; [captured output](evidence/acceptance.txt) |
| TLS | Validated certificates and IP identity for all three public endpoints |
| Public client traffic | Exact 200-record round trip over SASL_SSL |
| Access isolation | Wrong password and writes outside `colors-` refused |
| Storage credential scope | Actual application credentials listed both app buckets; backend bucket HeadBucket denied with HTTP 403 |
| Controlled broker outage | Stopped node 2, which led partition 0; all 100 pre-outage records survived |
| Partition recovery | Writable 1 second after `docker stop` completed |
| Broker rejoin | Node 2 registered again with zero lag and matching log end offsets |
| Consumer offsets | Every checked partition retained its committed offset across the outage |
| Controller restart | Node 1 re-authenticated and rejoined |
| Data across repeat converge | Separate 50-record topic remained an exact match; [record manifest](evidence/continuity.json) |
| Default destruction guard | Delete refused with exit 2 before destructive stages |
| Full delete and repeated delete | Both passed using published pins |
| Independent final resource inventory | Zero remaining resources; all recorded EBS volumes absent |

The separate continuity topic was `colors-continuity-20260910`. Its 50 records
were produced and read back before the published-pin converge, then read back
and compared exactly afterwards. The expected SHA-256 is
`5cc2ef77184dc7ff31424088d6a7bd955906bd6cfd284cb7f34631230c26e04a`.

## Performance observation and scope

The unthrottled burst workload sent 20,000 1 KiB records: 1,530.57 records/s
(1.49 MB/s), 5,785.23 ms average latency, and 11,042 ms p99. These are the
producer tool's measurements for a short burst following fault tests, not a
steady-state capacity benchmark. The high latency warrants workload-specific
testing and tuning before production use.

This proves functional deployment, authenticated traffic, persistence across
converges, and recovery from controlled container stops/restarts in one AZ.
It does not prove abrupt process-kill, disk-loss, AZ-loss, or transactional
recovery, and does not establish production availability or latency objectives.
The 1-second figure starts after the stop command returns; it is not a general
crash-recovery time guarantee.

## Issues found and fixed

1. Green compute coordination attempted to serialize workflow callbacks.
   Persisted coordinator options now contain only required backend identity fields.
2. New S3-backed OpenTofu state can be a synthetic empty snapshot with no lineage.
   The library accepts that exact virgin form only after independently proving
   remote state absence; failed pre-apply recovery remains guarded by provider
   absence checks.
3. Kafka CLI processes inherited the server's 2 GiB minimum heap, causing an
   observed OOM kill. CLI and healthcheck processes now use a 256 MiB maximum
   heap; existing brokers update one at a time with readiness checks. Both full
   successful runs completed after this fix, with no further observed OOM kill.
4. Acceptance assertions could accept stale/duplicate records or ignored fault
   command failures. Checks now compare current-run records, validate stop/start
   results, compare offsets per partition, enforce deadlines, and restore a
   stopped broker on failure or interruption.
5. Blue's copied launcher duplicated the compute dependency pin. It now inherits
   that pin from its immutable AutoMQ package dependency.

Final AutoMQ verification: Green 52 tests / 166 assertions, Red 60 tests plus
typecheck, Blue 117 tests, parity across all three ports, three golden fixtures,
shell syntax and 16 launcher checks. colors-compute validation included Blue 510,
Green 107 / 1,327 assertions, Red 314 plus typecheck, and 476 parity cases per colour.

## Resource lifecycle

The [before-delete inventory](evidence/resources-before-delete.json) records the
owned machines, exact EBS volume IDs, VPC dependencies, keypair, IAM user and all
three S3 buckets. All root volumes had `DeleteOnTermination=true`.

Full deletion completed successfully: host cleanup 129.397 s, application
storage 15.214 s, compute 301.940 s, backend finalization 24.407 s. All three
containers were independently confirmed absent before application storage was
removed. The S3 backend bucket was deleted last, including its versioned state.

A repeated delete succeeded with the backend already absent, taking the
`start -> backend-finalize` route without requiring SSH access.

The [final inventory](evidence/resources-after-delete.json) reports zero
remaining resources and zero remaining billable resources, with all three
recorded EBS volumes absent. The VPC and its default/untagged dependencies,
provider keypair, application IAM identity and all three buckets are gone.
Terminated EC2 history remains visible through the AWS API; it is not a running
resource. The owned local private/public SSH key files and profile aliases were
also confirmed absent.

To repeat the read-only audit for this recorded lifecycle, load the deployment
credentials and run `python3 evidence/check_resources.py`. Its baseline contains
the exact volume and VPC IDs from this run.

**Final state: no live AWS resources remain for this deployment.**

Only non-secret desired state and evidence are committed. Credentials, private
keys, Terraform working data and private run logs remain excluded from git.
