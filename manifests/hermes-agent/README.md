# hermes-agent

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (MIT, Nous Research) running as a Telegram-only gateway against the Anthropic API.

- **Image:** `docker.io/nousresearch/hermes-agent:v2026.6.5` (pinned by digest in `deployment.yaml`)
- **Model:** `anthropic/claude-opus-4.6` (Anthropic API key required — pay-as-you-go console, **not** a Claude.ai subscription)
- **Telegram mode:** long polling, outbound only — no ingress, no port exposed
- **State:** single PVC at `/opt/data`, 10 Gi, `local-path`
- **Sandboxing:** `terminal.backend: local` — tool subprocesses run inside this pod. On Talos there is no host shell, so the pod **is** the sandbox

## First-time setup

### 1. Create a Telegram bot

In Telegram, message [@BotFather](https://t.me/BotFather):

```
/newbot
```

Choose a display name and a username ending in `bot`. BotFather replies with an API token like `123456789:ABCdefGHIjklMNOpqrSTUvwxYZ`.

### 2. Find your numeric user ID

Message [@userinfobot](https://t.me/userinfobot) — it replies with your user ID (a number, not your @username).

### 3. Get an Anthropic API key

[console.anthropic.com → API Keys](https://console.anthropic.com/settings/keys). This is the pay-as-you-go API product, separate from a Claude.ai subscription.

### 4. Populate the Secret

The Secret in git ships with empty placeholder values. ArgoCD has `ignoreDifferences` on `/data` so it will not overwrite your real values. Create them with `kubectl`:

```sh
kubectl create secret generic hermes-agent-secrets \
  --namespace hermes-agent \
  --from-literal=ANTHROPIC_API_KEY='sk-ant-...' \
  --from-literal=TELEGRAM_BOT_TOKEN='123456789:ABCdef...' \
  --from-literal=TELEGRAM_ALLOWED_USERS='123456789' \
  --dry-run=client -o yaml | kubectl apply -f -
```

(`TELEGRAM_ALLOWED_USERS` is comma-separated for multiple users — keep it short to prevent strangers chatting up your bot at your billing's expense.)

### 5. Sync ArgoCD

The first sync creates the namespace, PVC, ConfigMap, ServiceAccount, placeholder Secret (immediately superseded by your `kubectl` apply in step 4 — order doesn't matter, but if you sync first the pod will start without credentials and reconcile after step 4). After the Secret is populated, bounce the pod so the init container re-materializes `/opt/data/.env`:

```sh
kubectl rollout restart deployment/hermes-agent -n hermes-agent
```

## Verify

```sh
# Pod should be Running, 1/1 ready
kubectl get pod -n hermes-agent

# Logs should show the Telegram connection
kubectl logs -n hermes-agent deployment/hermes-agent | grep -i telegram

# Exec into the pod to check Hermes' own view of the gateway
kubectl exec -n hermes-agent deployment/hermes-agent -- hermes gateway status
```

Then in Telegram, message the bot you created. You should get an agent reply within a few seconds.

## Rotating the bot token or API key

```sh
kubectl create secret generic hermes-agent-secrets \
  --namespace hermes-agent \
  --from-literal=ANTHROPIC_API_KEY='<new-key>' \
  --from-literal=TELEGRAM_BOT_TOKEN='<new-or-same-token>' \
  --from-literal=TELEGRAM_ALLOWED_USERS='<ids>' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/hermes-agent -n hermes-agent
```

The init container rewrites `/opt/data/.env` from the live Secret on every boot.

## Rollback

This deployment is idempotent. To roll back to a prior image version:

1. Look up the prior multi-arch digest on https://hub.docker.com/r/nousresearch/hermes-agent/tags
2. Edit `deployment.yaml`, swap the image tag + digest, commit, push.
3. ArgoCD syncs; the Recreate strategy terminates the running pod before the new one mounts the PVC.

To remove the workload entirely: `kubectl delete -f apps/workloads/hermes-agent.yaml`. The PVC stays — delete it with `kubectl delete pvc -n hermes-agent hermes-agent-data` if you also want to wipe state.

## What's NOT enabled (phase 2)

- **Web dashboard.** Set `HERMES_DASHBOARD=1` + Basic auth env vars and add a Service. Bind to `127.0.0.1` for `kubectl port-forward`, or `0.0.0.0` if you want to reach it via the `tailscale-subnet-router` route.
- **OpenAI-compatible API server.** Default-off (`API_SERVER_ENABLED=true` to enable). Not needed for Telegram-only use.
- **Stricter sandbox.** Talos already removes the "escape to host" axis, but if you want VM-grade isolation switch `terminal.backend` to `modal` (paid, external) or `daytona`.
- **Egress NetworkPolicy.** Hermes' web/search tools need broad outbound by design, so a strict allowlist conflicts with normal use. If you want one, write a `CiliumNetworkPolicy` with `toFQDNs` covering `api.anthropic.com` + `api.telegram.org` + DNS, and accept that the agent loses generic web access.
- **`readOnlyRootFilesystem`.** s6-overlay writes to `/run/service/*` and `/var/run/s6/*`. Adding `emptyDir` tmpfs mounts there and retesting is the path.

## How the config gets in

`config.yaml` and `.env` are both materialized into `/opt/data/` by the `seed-config` init container on every pod boot:

- **`config.yaml`** comes from the `hermes-agent-config` ConfigMap — git is the source of truth, edits inside the pod do not survive a restart.
- **`.env`** is built one `KEY=value` line per non-empty file in the `hermes-agent-secrets` Secret mount.

This means: to change a non-secret setting, edit `manifests/hermes-agent/configmap.yaml` and let ArgoCD sync. To change a credential, follow the rotation instructions above.

## References

- Hermes Agent docs: https://hermes-agent.nousresearch.com/docs/
- Telegram setup: https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram
- Docker layout: https://hermes-agent.nousresearch.com/docs/user-guide/docker
- Config schema: https://github.com/NousResearch/hermes-agent/blob/main/cli-config.yaml.example
