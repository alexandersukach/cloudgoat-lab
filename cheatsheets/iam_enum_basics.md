# iam_enum_basics — Cheatsheet

Bare commands for the CloudGoat `iam_enum_basics` scenario. Resource names carry a
per-deployment suffix (e.g. `cg-bob-cgidutxci3g3i0`) — grep `authorization.json`
for the exact names, or set them as variables below.

```bash
# --- vars (fill from your deployment) -----------------------------------
PROFILE=bob
ACCOUNT_ID=<your-account-id>
SUFFIX=<deployment-suffix>     # e.g. cgidutxci3g3i0
```

## Foothold
```bash
aws configure --profile "$PROFILE"          # paste bob's keys from start.txt
aws sts get-caller-identity --profile "$PROFILE"
```

## Enumerate everything, then grep for flags
```bash
aws iam get-account-authorization-details --profile "$PROFILE" > authorization.json
grep -n 'flag' authorization.json           # surfaces 4/5 flag ARNs + names
```

## Flag 1 — managed policy (in its Description)
```bash
aws iam get-policy \
  --policy-arn "arn:aws:iam::$ACCOUNT_ID:policy/cg-flag1-managed-policy-$SUFFIX" \
  --profile "$PROFILE"
```

## Flag 2 — inline policy on bob (in the Sid)
```bash
aws iam get-user-policy \
  --user-name "cg-bob-$SUFFIX" \
  --policy-name "cg-flag2-inline-policy-$SUFFIX" \
  --profile "$PROFILE"
```

## Flag 3 — group (in the Path)
```bash
aws iam get-group \
  --group-name "cg-flag3-group-$SUFFIX" \
  --profile "$PROFILE"
```

## Flag 4 — role (in the tags)
```bash
aws iam get-role \
  --role-name "cg-flag4-role-policy-$SUFFIX" \
  --profile "$PROFILE"
# trust policy lets bob assume it, but the role only has iam:ListUsers — dead end.
```

## Flag 5 — managed policy default version v1 (in the S3 Resource ARN)
```bash
aws iam get-policy-version \
  --policy-arn "arn:aws:iam::$ACCOUNT_ID:policy/cg-flag1-managed-policy-$SUFFIX" \
  --version-id v1 \
  --profile "$PROFILE"
```

## Teardown (always)
```bash
cloudgoat destroy iam_enum_basics    # run inside the CloudGoat container
```
