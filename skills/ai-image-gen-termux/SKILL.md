---
name: ai-image-gen-termux
description: AI image gen on Termux. Colab SD is the free pro route.
---

# AI Image Generation on Termux

Hermes itself cannot generate images — it's a text model. Any image work requires calling an external model. On Termux, three routes exist; only one is reliably free and high-quality.

## Route Selection (use in this order)

| Route | Free | Quality | Verdict |
|---|---|---|---|
| **AWS Bedrock (Titan/SDXL/Nova)** | ✅ $200 / 6 mo | ⭐⭐⭐⭐⭐ | **Best on Termux — pure-Python boto3** |
| **Google Colab + SD-XL** | ✅ T4 GPU | ⭐⭐⭐⭐⭐ | Free unlimited, needs Colab session |
| **Pollinations.ai** | ✅ no key | ⭐⭐ | Quick drafts only |
| **Leonardo.ai** | 150 credits/day | ⭐⭐⭐⭐ | Signup + web |
| **OpenRouter image models** | ❌ needs billing | ⭐⭐⭐⭐⭐ | 'Missing Authentication header' even with key |
| **Gemini API direct (image)** | ❌ paid only | ⭐⭐⭐⭐⭐ | 'Quota exceeded' on free tier |
| **Stable Diffusion local** | - | - | Won't run on Termux (needs GPU) |

**Rule:** When user asks 'can you generate an image', answer honestly — Hermes is a text model, you call an external service. Do not pretend to synthesize images yourself.

## AWS Bedrock Route (Best for Termux — pure Python)

boto3 is pure Python and installs on Termux cleanly. AWS CLI v2 install script fails on Termux (needs glibc) — **do not use the AWS CLI path, use boto3 directly**.

### Setup

```bash
pip install boto3
# Save creds to ~/.aws/credentials (chmod 600)
```

Credentials file:
```
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

### Free tier

- $200 credit / 6 months (new AWS accounts)
- Cost varies by model (check Bedrock pricing page — Titan was $0.01/image, Nova Canvas is higher)
- **Model deprecation check first:** Bedrock has retired multiple image models. As of 2026-10, `amazon.titan-image-generator-v1`/`v2:0` return `ResourceNotFoundException: model version has reached the end of its life`, and `stability.stable-diffusion-xl-v1`/`v0` are also EOL.
- **Recommended model:** `amazon.nova-canvas-v1:0` (Amazon's current image model). Requires access request in Bedrock console.
- **Discovery via SDK:** `boto3.client('bedrock').list_foundation_models()` returns all approved models. The correct filter parameter is one of `byProvider`, `byCustomizationType`, `byOutputModality`, `byInferenceType` (NOT `by='modelType'`). Iterate `modelSummaries` and filter `capabilities` for `'ImageGeneration'`.
- Regions: `us-east-1` (N. Virginia) or `us-west-2` (Oregon)

### Generate

```python
import boto3, base64, json
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
body = json.dumps({"promptText": prompt, "numberOfImages": 1, "imageConfig": {"dimension": "1024x1024"}})
resp = bedrock.invoke_model(modelId='amazon.nova-canvas-v1:0', body=body)
data = json.loads(resp['body'].read().decode('utf-8'))
with open('out.png', 'wb') as f:
    f.write(base64.b64decode(data['images'][0]['b64Image']))
```

Note: `amazon.nova-canvas-v1:0` uses `imageConfig.dimension` (`512x512`, `768x768`, `1024x1024`, `1024x1536`, `1536x1024`) instead of the older Titan/SDXL `quality` param. Older `quality: "standard"` param is not accepted on Nova.

### IAM User creation (5 min, before requesting model access)

1. IAM → Users → Add user → username `bedrock-images`
2. Permissions: **Attach policies directly** (not "Add user to group")
3. Search `AmazonBedrockFullAccess` (or `AdministratorAccess` for all-in-one)
4. Create user → Security credentials → Create access key
5. Use case: `Local code` (not CLI) — accepts warning about AWS Toolkit alternative (Termux has no AWS Toolkit)
6. **Do NOT provide Console access** unless user wants web login as that user

Bedrock Console → Model access → Request `amazon.nova-canvas-v1:0` (and any others). Approval is 5-10 min.

### Pitfalls

- **Model access takes 5-10 min to approve** after request — test will fail with `AccessDenied` until approved.
- **Never ask user to paste keys in chat** — the AWS Agent Toolkit setup instructions explicitly forbid this; have them write to `~/.aws/credentials` directly. If they already pasted, save immediately and treat as potentially compromised (rotate from AWS Console).
- **Prefix check on keys:** `AKIA...` = permanent access key (correct for scripts). `ASK...` = **temporary session credential** (expires in hours, needs session token). If user pastes an ASK key, they're copying from the AWS console header instead of the IAM user security-credentials page — point them to the right place.
- **Model end-of-life is silent in boto3** — the SDK returns `ResourceNotFoundException`, not a deprecation notice. Check model status on `bedrock.amazonaws.com` or by attempting a test invoke before writing a generate script.
- Working script: `~/gen_image.py`, output: `~/Hermes_Almari/04_AI_IMAGE_GEN/aws_generated/`. Credentials at `~/.aws/credentials` (also mirrored to `~/Hermes_Almari/99_COLLABORATION/aws/aws_credentials.env`).

## Free Fast Route: Pollinations.ai

```bash
curl -sL "https://image.pollinations.ai/prompt/<url-encoded-prompt>?width=1920&height=1080&nologo=true&seed=<int>" \
  -o out.jpg --max-time 180
```

- Only `sana` model, mediocre quality (user rejected it).
- **Pitfall:** Retries often return 2-byte files (rate limit / timeout). Retry loop with `sleep 10` between, verify size >10KB. On 3 failures move on — don't hammer.
- Batch wrapper: `~/gen_images.sh prompts.txt out_dir/` (one prompt per line).

## Pro Route: Colab + Stable Diffusion XL

Generate a self-contained Colab notebook, have the user paste + run. Free T4 GPU, unlimited images, professional quality.

### Minimal notebook cell

```python
!pip install -q diffusers transformers accelerate
from diffusers import StableDiffusionXLPipeline
from diffusers import EulerDiscreteScheduler
import torch

pipe = StableDiffusionXLPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    torch_dtype=torch.float16, use_safetensors=True,
).to("cuda")
pipe.scheduler = EulerDiscreteScheduler.from_config(pipe.scheduler.config)

def gen(prompt, seed=42, steps=30, cfg=6.0):
    g = torch.Generator("cuda").manual_seed(seed)
    return pipe(prompt, num_inference_steps=steps, guidance_scale=cfg, generator=g)

# For batch: loop over a prompts list, save each to ./outputs/
```

Save output folder via `google.colab.files.download` (zip).

### Prompt engineering for historical / cinematic scenes

Working modifier stack (append to base scene description):
```
<subject>, cinematic, photorealistic, epic, detailed, dramatic lighting, 8k, masterpiece
```

For historical scenes include: era, location, style anchor (`medieval painting`, `documentary photograph`, `period drama film still`). Include seed for reproducibility across shots.

**Pitfall:** Long prompts (>50 tokens) sometimes get silently truncated by SD-XL. Keep under ~40 tokens; use `negative_prompt` for unwanted elements rather than stuffing exclusions into positive prompt.

## Integration with Slideshow Video

Feed generated images to `ffmpeg` for zoom/blur slideshow (see `anime-vlog-workflow` skill for the exact zoompan recipe).

## Environment Notes

- **`/tmp/` writes fail on Termux** — always use `$HOME/` paths for output.
- **AWS CLI v2 install script requires glibc** — Termux uses bionic libc. `curl awscli | bash` will download but the resulting binary won't execute. Use `pip install boto3` directly instead; boto3 is pure Python and works.
- **boto3 client creation succeeds without credentials** (returns `BedrockRuntime` object) — only actual `invoke_model` calls fail without creds. Don't mistake a successful `boto3.client(...)` for auth being configured.
- **Gemini API image gen requires paid tier** — free-tier key returns `Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests` on `gemini-*-image` endpoints. Don't retry; that's a billing-tier block, not a transient error.
- **OpenRouter image models require billing** — even with a valid `OPENROUTER_API_KEY`, `/chat/completions` calls to `google/gemini-*-image`, `openai/gpt-5-image*` return `Missing Authentication header` unless the account has billing enabled. Free OpenRouter keys only work for text models.
- **Verify image validity by size** — corrupt/failed downloads come back as 2-100 byte files, not valid JPEGs. Check `os.path.getsize() > 10000` before treating as success.
- Working image-gen dir: `~/Hermes_Almari/04_AI_IMAGE_GEN/`.

## Files

- `~/gen_images.sh` — Pollinations batch wrapper
- `~/Hermes_Almari/04_AI_IMAGE_GEN/` — output dir
