# AWS Bedrock Setup for Termux (2026-10 verified)

Deep-dive for the AWS Bedrock image-gen route. `SKILL.md` carries the standing rules; this file has the full setup walkthrough and current model status.

## IAM User Setup (5 min)

1. Console: https://console.aws.amazon.com/iam/
2. **Users** → **Add user** → username `bedrock-images`
3. **Access key** step: select `Local code` (Termux Python scripts). The console suggests AWS Toolkit IDE — ignore, Termux has no IDE integration.
4. **Permissions** step: choose **Attach policies directly** (NOT "Add user to group" — no groups to add to on first setup). Search `AmazonBedrockFullAccess` (image only) or `AdministratorAccess` (all services, simpler).
5. Create user → **Security credentials** tab → **Create access key** → use case `Local code` → add any description → **Create**.
6. Copy `Access Key ID` (starts `AKIA`) and `Secret Access Key` (40 chars). Save immediately — secret only shown once.

## Root User vs IAM User

AWS new sign-ups land on the **Root user** console. Root can create IAM users. Password is what user set at signup — if forgotten, use "I don't remember my password" link (sends email reset link). Account ID (12 digits) is visible at top-right account icon on any AWS console page.

## Credentials File Format

```ini
# ~/.aws/credentials (chmod 600)
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
region = us-east-1
output = json
```

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json
```

Both files must be `chmod 600` — user-owned, no group/world access.

## Model Access Request

1. Bedrock Console: https://console.aws.amazon.com/bedrock/
2. Region top-right: `us-east-1`
3. **Model access** → **Manage model access**
4. Search `nova-canvas` → Request access
5. Wait 5-10 min for approval (automated)

Verify via SDK:
```python
import boto3
bedrock = boto3.client('bedrock', region_name='us-east-1')
r = bedrock.list_foundation_models()
for m in r['modelSummaries']:
    if 'ImageGeneration' in m.get('capabilities', []):
        print(m['modelId'])
```

## Current Model Status (2026-10)

| Model ID | Status | Notes |
|---|---|---|
| `amazon.nova-canvas-v1:0` | ✅ Current | Amazon's active image model. Uses `imageConfig.dimension`. |
| `amazon.titan-image-generator-v1` | ❌ EOL | Returns `ResourceNotFoundException` |
| `amazon.titan-image-generator-v2:0` | ❌ EOL | Returns `ResourceNotFoundException` |
| `stability.stable-diffusion-xl-v1` | ❌ EOL | Returns `ResourceNotFoundException` |
| `stability.stable-diffusion-xl-v0` | ❌ EOL | Returns `ResourceNotFoundException` |

**Never hard-code a model ID in a script without checking** — Bedrock retires models silently. Run `list_foundation_models()` before writing generate logic.

## Request Body Differences

Nova Canvas (current):
```json
{"promptText": "...", "numberOfImages": 1, "imageConfig": {"dimension": "1024x1024"}}
```

Older Titan/SDXL (deprecated):
```json
{"promptText": "...", "numberOfImages": 1, "quality": "standard"}
```

The `quality` param is NOT accepted on Nova Canvas — use `imageConfig.dimension` (`512x512`, `768x768`, `1024x1024`, `1024x1536`, `1536x1024`).

## boto3 SDK Quirks on Termux

- `boto3.client('bedrock-runtime', region_name=...)` succeeds WITHOUT credentials — only `invoke_model` fails. Don't treat a successful client creation as proof of auth.
- `list_foundation_models` filter params: `byProvider`, `byCustomizationType`, `byOutputModality`, `byInferenceType` (NOT `by='modelType'`, NOT `byOutputModality='IMAGE'` as a value).
- Client creation is region-independent, but invoke_model requires region match with where model access was requested.

## Verification Checklist

- [ ] `boto3` installed (`pip install boto3`)
- [ ] `~/.aws/credentials` exists, `chmod 600`
- [ ] Access Key ID starts with `AKIA` (not `ASK` — session creds)
- [ ] `boto3.client('sts').get_caller_identity()` returns Account ID
- [ ] `list_foundation_models()` shows `amazon.nova-canvas-v1:0`
- [ ] Test `invoke_model` with 1x512x512 prompt succeeds
- [ ] Output image size > 10KB (small files = corrupt/error payload)
