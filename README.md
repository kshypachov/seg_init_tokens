# token-initializer for Trembita 2.0 Proxy (Kubernetes Integration)

This utility is used for automated login to tokens (software or HSM) in the **Trembita 2.0 Proxy** component deployed in **Kubernetes**.

## 📦 Description

The utility reads the `UXP_TOKENS_PASS` environment variable, which must contain a comma-separated list of `token-id:password` pairs, such as:

```
UXP_TOKENS_PASS="0:1233,ciplus-78-5:##user##pass,ciplus-89-1109442605:12345678"
```

Each pair is separated by a **comma**, and the token ID and password are separated by a **colon**.  
⚠️ **Do not use commas or colons in token passwords.**

### How it works

1. If `UXP_TOKENS_PASS` is not set, the utility exits with an error.
2. If set, the string is split into key:value pairs and processed sequentially.
3. For each pair:
   - Runs [`token_login`](https://github.com/kshypachov/token_login) with `-w <id> <password>` to set the password.
   - If it fails, the utility enters **infinite wait** to prevent further retries.
4. Then performs 10 iterations:
   - Runs [`trembita-healthcheck`](https://github.com/kshypachov/trembita-healthcheck) to verify the proxy is operable.
   - Runs `token_login -r <id>` to verify the token accepted the password.
   - Any failure → enters infinite wait.

After all tokens are processed:
- If the last `trembita-healthcheck` fails → infinite wait.
- If successful → exits with code 0.

## ♻️ Why infinite waiting?

This utility **never retries failed passwords**.  
Some HSMs allow **only 5 incorrect attempts** before locking. To avoid this, the program enters an infinite sleep if an error occurs — preventing Pod restart loops that could exhaust the retry limit.

## ☸️ Usage in Kubernetes

This utility is typically used in a `startupProbe` to delay service readiness:

```yaml
startupProbe:
  exec:
    command:
      - token-initializer
  timeoutSeconds: 300
  initialDelaySeconds: 20
  periodSeconds: 20
  failureThreshold: 1000000
```

- The `failureThreshold` is set to `1000000` to allow up to ~11 days before the Pod is marked as failed.
- If something goes wrong, your monitoring tools will have **plenty of time** to alert you.

> Bonus: with 4 startup attempts per failure, you have about **1.5 months** before an HSM with a 5-attempt limit is locked out.

## ✅ Exit Codes

| Code | Meaning              |
|------|----------------------|
| `0`  | All tokens logged in |
| `1`  | Error occurred       |

## 🔗 Dependencies

- [`token_login`](https://github.com/kshypachov/token_login)
- [`trembita-healthcheck`](https://github.com/kshypachov/trembita-healthcheck)

Make sure both are available in the container PATH.

## 📄 License

MIT or other, if applicable.
