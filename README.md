# vita-tinfoil-config

Public deployment manifest for the Vita Aubrai chat [Tinfoil](https://docs.tinfoil.sh/containers/overview) enclave. This repository contains exactly one meaningful file — [`tinfoil-config.yml`](./tinfoil-config.yml) — which pins the precise `sha256` digest of the Docker image that Tinfoil verifies at deploy time.

## Why does this repo exist?

VitaDAO's main app code lives in a **private** repo ([`VitaDAO/vita-app`](https://github.com/VitaDAO/vita-app)). That's where the proxy source, Dockerfile, and GitHub Actions workflow all live, under [`tinfoil-proxy/`](https://github.com/VitaDAO/vita-app/tree/main/tinfoil-proxy).

Tinfoil, however, requires the `tinfoil-config.yml` deployment manifest to live in a **public** repository so that anyone can independently audit exactly what's running inside the enclave. Splitting the config into this tiny public repo lets us keep the main app code private while still giving outside observers a clear, immutable trust anchor.

## What to look for

When the Aubrai chat client establishes an HPKE session with the Tinfoil enclave, it performs remote attestation against the values pinned in [`tinfoil-config.yml`](./tinfoil-config.yml). If anything about the deployed image changes — a different digest, a different shim config, a different set of secrets — the verification fails and the session is refused by the browser before any user data is sent.

Specifically:

- **`image`** — the full OCI reference, including the `@sha256:...` digest. This is what Tinfoil pulls and what its attestation cryptographically binds to.
- **`shim.paths`** — the allow-list of HTTP paths the enclave exposes. Anything not in this list is rejected by Tinfoil's reverse-proxy shim before even reaching the container.
- **`secrets`** — the *names* of secrets the enclave needs (values are sealed under the enclave measurement and set in the Tinfoil dashboard, never checked into any repo).

## What's actually inside the enclave

The image referenced by the pinned digest is a Deno 2 HTTP server that proxies three endpoints:

| Route | What it does |
|---|---|
| `POST /api/aubrai-x402-chat`    | Bearer-auth + consent gates + tier quota + x402 USDC payment + forward to `x402-api.aubr.ai` |
| `POST /api/supermemory-context` | Bearer-auth + forward search query to `api.supermemory.ai/v4/profile` |
| `POST /api/aubrai-x402-status`  | Stateless polling proxy for `x402-api.aubr.ai/api/chat/status/{id}` |
| `GET  /health`                  | Liveness probe (no auth) |

The source for these handlers lives in `vita-app/tinfoil-proxy/src/` (private). The build output is reproducible from that source via `vita-app/.github/workflows/build-tinfoil-proxy.yml` and is published to the **public** GHCR package [`ghcr.io/vitadao/vita-tinfoil-proxy`](https://github.com/VitaDAO/vita-app/pkgs/container/vita-tinfoil-proxy), so anyone can pull the image, inspect its layers, and confirm the digest matches what's pinned here.

## Trust model

1. **Browser** (`SecureClient` SDK from `tinfoilsh/tinfoil-js`)
   ↓ HPKE-encrypts request body against the enclave's attested public key
2. **Tinfoil shim** (on the enclave host)
   ↓ Terminates TLS, forwards the still-HPKE-encrypted body onto the upstream port
3. **Deno proxy** (inside an AMD SEV-SNP / Intel TDX confidential VM)
   ↓ Only now is the body decrypted, inside hardware-isolated memory the host OS cannot read
4. **Upstream service** (Supabase, x402-api.aubr.ai, api.supermemory.ai)
   ↓ Sees plaintext — because at that point the request is leaving the enclave anyway

Neither VitaDAO, nor the Tinfoil hosting infrastructure, nor any network intermediary, nor even a root-privileged process on the enclave host can read the request body while it's in transit or while it's being assembled inside the proxy. The only actors who can read the prompt are the browser that sent it and the attested enclave that processes it.

## How to verify a release

Any observer can independently verify that a given Tinfoil deployment matches a given `v*.*.*` tag of this repo:

1. Check out the tag in this repo: `git checkout v0.1.0`
2. Read the digest from `tinfoil-config.yml`: `grep image tinfoil-config.yml`
3. Pull the image from GHCR: `docker pull ghcr.io/vitadao/vita-tinfoil-proxy@sha256:<digest>`
4. Inspect its layers: `docker image history ghcr.io/vitadao/vita-tinfoil-proxy@sha256:<digest>`
5. Cross-reference with the matching commit in `VitaDAO/vita-app` (the commit referenced in the release notes), whose `tinfoil-proxy/` subtree is the sole build input for the image.

## Release history

See the [Releases tab](https://github.com/VitaDAO/vita-tinfoil-config/releases) for every deployment pinned here, with the matching digest and source commit of each.
