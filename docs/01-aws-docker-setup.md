# Hosting the Wazuh manager on AWS + CloudTrail integration

Covers launching an EC2 instance sized correctly for the full Wazuh
stack, and wiring up CloudTrail so Wazuh monitors your AWS account's own
activity, not just endpoint agents.

---

## Part 1 — Launching a properly-sized instance

### Lessons learned the hard way

- **1GB RAM (`t2.micro`/`t3.micro`) is not enough.** The indexer's JVM
  alone tries to reserve ~1GB heap by default — on a 1GB total instance
  this fails immediately with `insufficient memory for the Java Runtime
  Environment`.
- **2GB RAM (`t3.small`) is borderline.** It can run the stack at rest,
  but under load (e.g. mid crash-loop-retry, or right after startup) it
  can still hit OOM kills severe enough to freeze SSH itself, not just
  the container.
- **~4GB RAM is comfortable.** Something like `c7i-flex.large` or
  `t3.medium` (if available in your region/account) gives real headroom
  with no tuning tricks needed.
- **20GB disk was not enough once fully initialized** — three service
  images plus data volumes filled it completely. Use 30GB+.
- **New/unverified AWS accounts may be capped to only the smallest
  instance sizes** ("not eligible under the Free Plan" tooltip on larger
  types), independent of region. Check **Service Quotas → Running
  On-Demand Standard instances** for a self-service increase request, or
  just pick from what's actually available to your account.
- **Newer AWS regions may not offer every instance size** even within a
  supported family (e.g. a region might support `t3.micro`/`t3.small` but
  not `t3.medium`). If your target size doesn't appear, check region
  availability before assuming it's an account issue.

### Launch checklist

- **Instance type:** something with ~4GB RAM (see above)
- **Storage:** 30GB+
- **Security group inbound rules:**
  | Port | Purpose | Source |
  |---|---|---|
  | 22 | SSH | My IP |
  | 443 | Dashboard | My IP |
  | 1514-1515 | Agent enrollment/events | Agent source IP(s), or 0.0.0.0/0 if agents are unpredictable |

**Note:** "My IP" is resolved once, at the time you create the rule — if
your home/ISP IP changes later, SSH will silently fail (looks identical
to the instance itself being unreachable). If SSH suddenly stops working
with no other symptoms, check your current IP first:
```bash
curl ifconfig.me
```
and compare it against the security group rule before assuming something
is wrong with the instance.

## Part 2 — Installing the Wazuh stack

```bash
sudo dnf install -y docker git findutils curl
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# log out and back in, or just prefix commands with sudo going forward
```

Docker Compose plugin often isn't present on a fresh Amazon Linux
instance — install the standalone binary instead:
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
Use `docker-compose` (hyphenated) throughout, not `docker compose`.

Set `vm.max_map_count` **before** starting anything — OpenSearch requires
this, and it's not set high enough by default:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Clone (pin to an actual version tag, not a literal placeholder):
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node
```

Generate the SSL certs — required before first start, or containers fail
to mount cert files:
```bash
sudo docker-compose -f generate-indexer-certs.yml run --rm generator
```
If this fails with `find: command not found`, install `findutils` first
and retry.

Start the stack:
```bash
sudo docker-compose up -d
```

## Part 3 — CloudTrail integration

This lets Wazuh monitor your AWS account's own activity (API calls, IAM
changes, security group edits, instance launches) — not just what's
happening inside a monitored host.

### 3.1 Create a CloudTrail trail

- AWS Console → **CloudTrail → Create trail**
- Storage: new S3 bucket, note the exact bucket name
- Management events: keep enabled (Read + Write)
- KMS encryption: optional. If enabled, you'll need extra IAM permissions
  (see 3.2) and a KMS key alias.

### 3.2 Create an IAM user with read + decrypt access

Custom policy (adjust bucket name; drop the KMS block entirely if you
didn't enable encryption):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "*"
    }
  ]
}
```

If using KMS, also check the **KMS key's own key policy** (separate from
the IAM policy above) explicitly grants this IAM user `kms:Decrypt` — a
commonly missed step, since KMS keys have their own resource-based policy
in addition to IAM's identity-based policy.

Generate access keys for this user (use case: "Application running on an
AWS compute service") and save both values.

### 3.3 Install boto3 and configure credentials on the manager

```bash
sudo docker exec -it <manager-container-name> bash
```

Inside the container:
```bash
dnf install -y python3-pip   # Amazon Linux base image
python3 -m pip install boto3

mkdir -p /root/.aws
cat > /root/.aws/credentials << 'EOF'
[default]
aws_access_key_id = <access-key-id>
aws_secret_access_key = <secret-access-key>
EOF

cat > /root/.aws/config << 'EOF'
[default]
region = <bucket-region>
EOF
```

**The region in `/root/.aws/config` matters even if you never explicitly
picked one** — boto3 defaults to `us-east-1` if unset, and will fail with
`IllegalLocationConstraintException` against a bucket in any other
region. This is an easy one to miss since the error message doesn't
obviously point at "you forgot to set a region."

### 3.4 Add the wodle to ossec.conf

Still inside the manager container:
```bash
nano /var/ossec/etc/ossec.conf
```

Add before the final `</ossec_config>`:
```xml
<wodle name="aws-s3">
  <disabled>no</disabled>
  <interval>10m</interval>
  <run_on_start>yes</run_on_start>
  <skip_on_error>yes</skip_on_error>
  <bucket type="cloudtrail">
    <name><bucket-name></name>
  </bucket>
</wodle>
```

**Double-check the closing tag is `</wodle>`, not `<wodle>`.** A missing
`/` here is a genuine, easy-to-make typo that silently breaks the entire
config — every daemon fails to start (`wazuh-db` in particular, with an
unhelpful "line 0" error) because the XML can't be parsed at all, not
because of anything wodle-specific.

Exit the container, restart just the manager:
```bash
sudo docker-compose restart wazuh.manager
```

### 3.5 Verify

```bash
sudo docker exec -it <manager-container-name> /var/ossec/wodles/aws/aws-s3 \
  --bucket <bucket-name> --type cloudtrail --debug 2
```

Running the wodle manually with `--debug 2` surfaces the real underlying
error (region mismatch, decrypt failure, etc.) far more clearly than the
generic "Unknown error / exit code 1" the scheduled run logs.

Check the dashboard: hamburger menu → **Cloud Security → Amazon AWS** (or
Threat Intelligence, depending on version). Widen the time range if
nothing shows up — data may exist but be outside the default window.

---

## Persistence — config edits inside vs. outside the container

Wazuh's Docker Compose setup uses **named Docker volumes**
(`wazuh_etc`, etc.), not simple host bind-mounts to the `config/` folder
in the repo. That folder is only used to *seed* the volume on first
creation — editing files there after the fact has no live effect on an
already-running stack.

Editing `/var/ossec/etc/ossec.conf` **directly inside the container**
(via `docker exec ... nano ...`) *does* persist correctly across
`docker-compose restart` and even a full `docker-compose down` /
`docker-compose up -d` cycle — because the named volume itself survives.
It's only lost if the volume is explicitly deleted with
`docker-compose down -v`.

`/root/.aws/credentials` and `/root/.aws/config`, however, are **not**
in a named volume — they live in the container's writable layer, which
is genuinely lost on container recreation (not on a plain restart). If
you ever run `docker-compose up -d --force-recreate` on the manager, or
tear down and rebuild the stack, these two files need to be recreated.

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `insufficient memory for the Java Runtime Environment`, mmap failure | Instance RAM too small for indexer's JVM heap | Resize instance up, or cap heap via `OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m` (band-aid, not a real fix below ~2GB) |
| Container OOM-killed (exit 137) even with heap capped | Total instance RAM still insufficient once OS + all 3 services are accounted for | Resize instance up |
| `No space left on device` | Disk too small once images + data volumes fill it | Resize EBS volume, then `growpart` + `xfs_growfs` |
| SSH suddenly stops working, no other symptoms | Security group's "My IP" rule is stale (your public IP changed) | `curl ifconfig.me`, compare to the rule, update if different |
| `IllegalLocationConstraintException` from the wodle | boto3 defaulting to `us-east-1`, bucket is elsewhere | Set `region` in `/root/.aws/config` inside the manager container |
| `wazuh-db did not start correctly`, `Error reading XML file ... (line 0)` | Usually a permissions issue on `ossec.conf`, not actually XML content | Check ownership matches other files in `/var/ossec/etc/` (should be `root:wazuh` or similar) — `docker cp` in particular tends to write files as `root:root`, breaking this |
| `mismatched tag` XML parse error | Genuine syntax error, e.g. `<wodle>` instead of `</wodle>` | Read the exact line/column from the error, check that specific tag pair |
