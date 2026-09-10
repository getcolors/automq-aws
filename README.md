# automq-aws

Live lifecycle testing passed; all deployment AWS resources have been deleted.

Three AutoMQ 1.7.4 brokers/controllers on AWS EC2, provisioned through
[colors-compute](https://github.com/getcolors/colors-compute).

Profile `automq-aws` uses three `t3.large` machines in `us-east-1a` and an image
pinned by digest. The deployment creates its VPC, SSH keypair, machines, S3
Terraform state bucket, separate S3 data and ops buckets, and an IAM identity
restricted to the two application buckets. Replication factor is 1: AutoMQ
stores durable records in object storage.

This AWS-only configuration uses IP-address certificates issued by a deployment
CA. It requires no DNS zone. Clients must trust the exported CA explicitly;
TLS certificate and IP verification remain enabled. The CA private key remains
on the issuer node. This is a live assessment configuration, with one AZ and
public SSH/Kafka ingress; it is not a production availability assessment.

## Run

Install Babashka, OpenTofu, Ansible, AWS CLI, OpenSSL, Python with boto3, and kcat.
The copied `green` launcher loads immutable published package dependencies.
Credentials are loaded from `../.envrc` using the `COLORS_PAR_AWS_ACCESS_KEY_ID`
and `COLORS_PAR_AWS_SECRET_ACCESS_KEY` variables. No credentials belong in git.

```bash
direnv allow
./green create
```

`create` converges infrastructure and runs on-host and external acceptance,
including authenticated produce/consume and a targeted broker outage. Repeat
`create` to converge the same deployment. Never run `build` during a converge:
`.colors/` contains the active scripts and inventory.

The public CA is exported to
`.colors/automq-aws/automq-acceptance/ca.crt`. Obtain the client password with
`ssh automq-aws sudo automq-credential`; pass the CA to kcat with
`-X ssl.ca.location=<path>` and use `SASL_SSL` with `SCRAM-SHA-512`.

## Delete

The desired state protects against destruction by default. To intentionally
delete the entire deployment, including all application records:

```bash
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

Deletion stops the brokers, removes application storage and its IAM identity,
destroys compute and its SSH keypair, and finally deletes the managed backend
bucket and its versioned state. Adopted external buckets are not used here.

See [verification.md](verification.md) for measured live results and final
resource status. Private logs and `.colors/` are excluded from git.

The launcher and `.agents/skills/package-automq-green/` were manually copied
from AutoMQ commit `debb50b`; no installer lockfile is claimed. The root launcher
is byte-identical to the copied payload and pins source commit `e3beeaf`.
