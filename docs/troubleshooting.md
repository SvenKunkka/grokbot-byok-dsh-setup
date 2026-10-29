# Troubleshooting

## Error codes

The pi-ai adapter fails with stable codes. They tell you which layer broke.

| Code | Meaning | Fix |
|---|---|---|
| `MISSING_CREDENTIAL` | `apiKeyEnv` resolved to empty | Fill the key in Settings → Models |
| `INVALID_CREDENTIAL` | Key present but rejected | Check the key, and that `baseURL` matches the provider |
| `UNKNOWN_MODEL` | Model id not in the route's catalog | Add it under `models:` or fix the id |
| `QUOTA` | Provider says you are out of credit | Top up, or switch the bot's model plan |
| `RATE_LIMIT` | Temporary throttle, retried per `retryPolicy` | Lower concurrency or raise retries |
| `UNSTORABLE_PROVIDER_ID` | Route key is not a lowercase-hyphen identifier, so it cannot be stored as a login | Use `apiKeyEnv` instead of interactive login for that route |
| `IMAGE_OFFLOAD_REQUIRED` | Base64 image payload exceeded `maxRequestImageBytes` | Raise the limit or drop the attachment |
| `UNSUPPORTED_OPTION` | `stop` sequences requested | pi-ai cannot guarantee them across providers |

## Common symptoms

### A route disappeared from the model picker

Almost always a **duplicate `llm-pi-ai` entry**. The `providers` map is replaced wholesale, not merged. Search your patch file:

```bash
grep -n "id: llm-pi-ai" ~/.dsh/profiles/<profile>/cordis.patch.yml
```

Expect exactly one hit.

### Save is rejected after a small edit

Editing only `displayName`, `apiKeyEnv` or `baseURL` while the route still has a models error is refused. The whole provider config must validate in one save. Fix or delete the broken model entry in the same edit.

### Hand-declared gateway fails to load

A route pi-ai does not know requires all three: `api`, `baseURL`, and a **non-empty** `models` list. Catalog routes can omit `api`; yours cannot.

### Wrong request shape on a custom gateway

Use `compat` to reshape the request rather than patching code:

```yaml
compat:
  thinkingFormat: deepseek        # how thinking levels are spelled
  # supportsMaxOutputTokens: false  # omit max_output_tokens (openai-responses)
```

### `dsh --profile desktop` refuses to boot

```
error: profile "desktop" is managed exclusively by the Electron application
```

The desktop profile belongs to the app. Copy it under a new name to boot from the CLI. Also note the argument order:

```bash
dsh --profile <name> --port 61999    # correct
dsh --profile <name> web             # error: too many arguments
```

### The page returns 401

Missing `?token=`. The token is printed once at boot:

```
dsh web: http://127.0.0.1:61999/?token=...
```

### `timeout` not found on macOS

macOS ships no GNU `timeout`. Use a background job and `kill`, or install coreutils.

## Reading keys safely

The credential store refuses read-back to agents and scripts:

```
The native credential store is not available to Agent commands
```

That is by design, not a bug. Never work around it by copying `.credentials.yaml`; enter keys only through the UI.
