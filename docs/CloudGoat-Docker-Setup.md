# CloudGoat Setup (Docker) on Kali

**Tool:** [RhinoSecurityLabs/cloudgoat](https://github.com/RhinoSecurityLabs/cloudgoat)
**Approach:** Docker (official prebuilt image)
**Host for these commands:** the Kali guest (see `Kali-VMware-Setup.md` for the VM)
**Companion:** `Kali-AWS-PenTest-Reference.md` (AWS CLI, profiles, attacker toolkit)
**Date:** 2026-06-03 (in progress)

---

## ⚠️ Read first — safety & cost

CloudGoat is a **"vulnerable by design"** AWS deployment tool. It intentionally
creates exploitable cloud infrastructure so you can practice attacking it.

- It deploys **real, billable AWS resources.** They cost money the whole time
  they exist.
- **Always use your own personal/lab AWS account — never work/production.**
- **Always `cloudgoat destroy` a scenario when you're done** to stop charges.
- Use a dedicated IAM user with its own access keys (easy to rotate/revoke).
- `cloudgoat config whitelist --auto` restricts the vulnerable resources to your
  current public IP so you're not exposing them to the whole internet.

---

## Why Docker (and why the requirements list looked contradictory)

CloudGoat's README documents **two alternative install paths**:

1. **Manual install** — you install everything on the host yourself. *This* is
   what the README "Requirements" list is for:
   - Python 3.9+, Terraform ≥ 1.5.0, AWS CLI, Azure CLI, jq
   - `sudo apt install terraform awscli azure-cli jq -y`
   - `pipx install cloudgoat`

2. **Docker install** — the image **already contains** Python, Terraform,
   AWS CLI, jq, and CloudGoat. You install **none** of those on Kali.

They are an **either/or**. The Requirements list isn't a prerequisite for the
Docker path — it documents the *other* path. We chose **Docker** because it
isolates Terraform/AWS-CLI versions inside the image so they can't get tangled
with the host (which is the kind of mess that caused trouble before).

### Chosen method: pull the official prebuilt image
This is what the README **and** the InfoSec Institute guide both recommend —
the simplest, most-supported path. `docker run` auto-downloads the image on
first use, so there is **nothing to build**:
```
docker run -it rhinosecuritylabs/cloudgoat:latest
docker run -it -v ~/.aws:/root/.aws/ rhinosecuritylabs/cloudgoat:latest
```

### Alternative (not used): build from a clone
You *can* instead `git clone` the repo and `docker build -t cloudgoat .`. That
gives the freshest code and puts scenario source on the host, but adds a build
step. We skipped it for simplicity. If you want the scenario walkthroughs on the
host for reading, you can still `git clone` the repo purely as reference — no
build needed (scenario files also exist inside the container at
`/usr/src/cloudgoat`).

---

## Do you need AWS CLI on the Kali host?

- **To deploy/destroy CloudGoat scenarios:** No — the container has AWS CLI.
- **To actually *do* the labs** (enumerate/exploit the deployed resources as the
  attacker): Yes — you run `aws`, `pacu`, etc. from the Kali host.
- **Clean approach:** install AWS CLI **v2** on the host and run
  `aws configure --profile cloudgoat` *once*. Because we mount `~/.aws` into the
  container, the host **and** CloudGoat read the same credentials. One config,
  works everywhere. (We install v2, not v1: v2 is the current version and AWS
  only ships it via their official installer — `apt`/`pip` give the old v1.
  On ARM use the **aarch64** build.)

---

## The Docker state gotcha (important)

The image's `WORKDIR` is `/usr/src/cloudgoat/`, its entrypoint is `/bin/bash`,
and **CloudGoat's Terraform state is stored inside the container.**

➡️ If you delete the container before destroying your scenarios, you lose the
state — and the billable AWS resources keep running with no easy teardown.

**Mitigation — use a *named, persistent* container (no `--rm`):**
- `--name cloudgoat` → reusable container that survives reboots.
- **Never `docker rm cloudgoat`** until `cloudgoat destroy all` is clean.
- Leave with `exit` (container stops, state preserved).
- Return with `docker start -ai cloudgoat` (same container, state intact).
- `-v ~/.aws:/root/.aws/` keeps credentials on the host so they persist even if
  the container is ever rebuilt.

---

## Steps

> ⚠️ **WHERE TO RUN WHAT — read this first.**
> **All `cloudgoat` commands** (`config`, `create`, `list`, `destroy`) run from
> **INSIDE the CloudGoat Docker container** — the `root@<id>:/usr/src/cloudgoat#`
> shell (note the `#`). `cloudgoat` does not exist on the Kali host.
> - Container prompt = `<id>:/usr/src/cloudgoat#` → run `cloudgoat` here ✅
> - Kali host prompt = `┌──(user㉿kali)-[~]` / `$` → `cloudgoat` NOT found here ❌
>
> Steps marked **(Host)** run on Kali; steps marked **(Container)** run inside the
> container. Attacker tools (`aws`, `pacu`, etc.) run on the **host**; deploying/
> destroying scenarios runs in the **container**.

### Step A — (Host) Install AWS CLI v2 + configure the `cloudgoat` profile
Install AWS CLI v2 (aarch64) — see **`Kali-AWS-PenTest-Reference.md` → AWS CLI v2**.
Then configure your personal AWS keys once on the host:
```bash
aws configure --profile cloudgoat
#   AWS Access Key ID     : <your key>
#   AWS Secret Access Key : <your secret>
#   Default region name   : us-east-1
#   Default output format : (press Enter)
```
This writes `~/.aws/credentials` + `~/.aws/config` on the host — which the
container reads via the mount in Step B.

### Step B — (Host) Launch the persistent container (auto-pulls the image)
```bash
docker run -it --name cloudgoat -v ~/.aws:/root/.aws/ rhinosecuritylabs/cloudgoat:latest
# first run downloads the image, then your prompt becomes
#   root@<id>:/usr/src/cloudgoat#   — you are now INSIDE the container
```
No separate clone/build needed — `docker run` pulls `rhinosecuritylabs/cloudgoat:latest`
from Docker Hub on first use.

### Step C — (Container) Point CloudGoat at your AWS profile + set whitelist
```bash
cloudgoat config aws
#   when prompted for the profile name, enter:  cloudgoat

cloudgoat config whitelist --auto   # restricts resources to your current IP
```

### Step D — (Container) Deploy / manage scenarios  ⏸️ COST STARTS HERE
> Pause and pick a cheap starter first. `iam_enum_basics` is a good low-cost one.
```bash
cloudgoat create iam_enum_basics    # deploy (billable!)
cloudgoat list deployed             # see what's running
cloudgoat destroy <scenario-name>   # tear down ONE scenario
cloudgoat destroy all               # tear down everything
```

---

## Attacker-side tooling

The tools you use to *exploit* deployed scenarios (Pacu, trufflehog, ScoutSuite,
CloudFox, S3 enumerators, enumerate-iam) live on the **Kali host**, not in the
CloudGoat container. Install + reasoning moved to
**`Kali-AWS-PenTest-Reference.md` → Attacker toolkit** (general AWS pentest kit,
not CloudGoat-specific).

## Scenario cost tiers (pick cheap while learning)

Cost driver = whether the scenario deploys **always-on, hourly-billed** resources.
- **~Free:** IAM, Lambda, SNS, SQS, S3 (tiny), Secrets Manager (pennies).
- **Hourly cost:** EC2, RDS, ECS/EFS, Elastic Beanstalk (runs EC2), Load Balancers, Glue.

**Cheapest starters (IAM/Lambda only — essentially $0):**
| Scenario | Difficulty | Deploys | Teaches |
|---|---|---|---|
| iam_enum_basics | Easy | IAM only | Enumerate a key's permissions (foundational) |
| iam_privesc_by_rollback | Easy | IAM only | Privesc via policy version rollback (classic 1st privesc) |
| iam_privesc_by_key_rotation | Easy | IAM only | Privesc via key/credential management |
| lambda_privesc | Easy | Lambda + IAM | iam:PassRole + Lambda role assumption |
| sns_secrets | Easy | SNS + API GW | Topic subscription → API key discovery |

**Avoid early (hourly EC2/RDS/ECS):** data_secrets, beanstalk_secrets,
cloud_breach_s3, ec2_ssrf, iam_privesc_by_attachment, ecs_*, rds_snapshot,
rce_web_app, codebuild_secrets, ecs_efs_attack, secrets_in_the_cloud.

**Recommended free progression:** `iam_enum_basics` → `iam_privesc_by_rollback`
→ `lambda_privesc`.

> `cloudgoat list all` shows all scenarios; `cloudgoat list deployed` shows what's
> currently running (and billing). **Always `cloudgoat destroy <scenario>` when done.**

## Troubleshooting

### `terraform init` fails: "could not connect to registry.terraform.io ... no such host"
**Cause:** DNS failure *inside the container*. It inherited the host's DNS server
(`172.16.208.2`, the VMware NAT resolver), which is flaky — a single lookup
(e.g. `whitelist --auto` → ifconfig.co) may succeed while Terraform's many rapid
lookups during `init` fail. Same DNS-flakiness family that prompted the VM rebuild.

**Immediate fix (in the container — preserves config.yml/whitelist):**
```bash
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" > /etc/resolv.conf
cat /etc/resolv.conf            # confirm public DNS
cloudgoat create <scenario>     # retry
```

**Durable fix (Kali host — applies to all containers):**
> Does NOT touch the host `/etc/resolv.conf` (host DNS is fine, leave it alone).
> It only creates `/etc/docker/daemon.json`, which controls *container* DNS.
> `:53` in the original error just means "the DNS port" — 53 is the standard DNS
> port; the error was saying the resolver at 172.16.208.2:53 didn't answer.

1. **(Container)** Leave gracefully — stops but preserves the container:
   ```bash
   exit
   ```
2. **(Host)** Make sure daemon.json doesn't already exist (don't clobber it):
   ```bash
   cat /etc/docker/daemon.json          # expect: No such file or directory
   ```
   If it instead prints JSON, merge by hand rather than overwriting.
3. **(Host)** Create it in nano (needs sudo — `/etc/docker` is root-owned):
   ```bash
   sudo nano /etc/docker/daemon.json
   ```
   Paste, then save with Ctrl+O / Enter / Ctrl+X. **Use a single line** to avoid
   nano's paste "staircase" indentation (JSON ignores whitespace, so this is valid):
   ```json
   {"dns": ["8.8.8.8", "1.1.1.1"]}
   ```
   Validate after saving: `jq . /etc/docker/daemon.json` (pretty-prints if valid;
   errors if malformed). Docker also refuses to start on invalid JSON, so a
   successful `systemctl restart docker` is itself confirmation.
4. **(Host)** Verify + restart Docker:
   ```bash
   cat /etc/docker/daemon.json
   sudo systemctl restart docker
   ```
5. **(Host)** Reattach (preserves config.yml + whitelist):
   ```bash
   docker start -ai cloudgoat
   ```
6. **(Container)** Verify DNS, then deploy:
   ```bash
   cat /etc/resolv.conf            # should now show nameserver 8.8.8.8
   cloudgoat create iam_enum_basics
   ```

**Fallback** (only if step 6 still shows `172.16.208.2`) — safe because nothing is
deployed yet: `exit` → `docker rm cloudgoat` → re-run the Step B `docker run ...`
→ redo `cloudgoat config aws` + `cloudgoat config whitelist --auto` → create.
(AWS creds persist via the `~/.aws` mount.)

## Quick reference — leaving & returning
```bash
exit                          # leave the container (state preserved)
docker start -ai cloudgoat    # come back to the same container
docker ps -a                  # list containers (running + stopped)
# Do NOT 'docker rm cloudgoat' until 'cloudgoat destroy all' is clean.
```

**Moving files between container and host (no cat/paste):**
```bash
# OUT of container → host   (syntax: docker cp <container>:<src> <hostdest>)
docker cp cloudgoat:/usr/local/lib/python3.12/site-packages/cloudgoat/<scenario_instance>/start.txt ~/start.txt
docker cp cloudgoat:/usr/local/lib/python3.12/site-packages/cloudgoat/<scenario_instance> ~/<scenario>   # whole folder
# INTO container ← host
docker cp ~/file cloudgoat:/path/
```
Scenario instances live at
`/usr/local/lib/python3.12/site-packages/cloudgoat/<scenario>_<random>/`
(contains start.txt, Terraform state, any solution files).

---

## Progress

- ✅ Decided on Docker route (official prebuilt image), with reasoning above
- ✅ Verified actual repo README + Dockerfile (state-in-container confirmed);
     cross-checked against InfoSec Institute guide (both recommend prebuilt pull)
- ✅ Step A — AWS CLI v2 on host + `aws configure --profile cloudgoat`
     (verified: `aws configure list-profiles` → `cloudgoat`)
- ✅ Step B — launched named container `cloudgoat` (image pulled, now inside at
     `/usr/src/cloudgoat#`). Run all `cloudgoat` commands from this container shell.
- ✅ Step C — `cloudgoat config aws` → profile `cloudgoat` saved;
     `cloudgoat config whitelist --auto` → whitelist.txt created with public IP
     69.242.74.230/32 (correct — that's the host's public IP, not the `ifconfig`
     private IPs)
- ✅ DNS fix — first `cloudgoat create` failed at `terraform init` (container
     couldn't resolve registry.terraform.io via flaky NAT DNS 172.16.208.2).
     Fixed durably via `/etc/docker/daemon.json` → `{"dns":["8.8.8.8","1.1.1.1"]}`
     + `systemctl restart docker`. Container resolv.conf now shows 8.8.8.8
     (`# Overrides: [nameservers]`), persists across restarts. Config/whitelist intact.
- ✅ Step D — deployed `iam_enum_basics` (10 resources, Apply complete). Scenario
     issued foothold creds for IAM user **bob** (`bob_access_key` / `bob_secret_key`).
     start.txt at `.../site-packages/cloudgoat/iam_enum_basics_cgidhfy2qa2fi9/`
     (contains only the creds — objective is intentionally open-ended).
     ⏳ User is working the scenario **solo** (wants to learn enumeration without
     walkthroughs) — give nudges only when asked, don't spoil.
     Teardown when done: `cloudgoat destroy iam_enum_basics`.

*(This doc is updated as we go.)*
