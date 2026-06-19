# authentik SAML Wildcard ACS URL Patch

This patch adds wildcard `*` support to the ACS URL field of the SAML provider.

## What it changes

| File | Change |
|------|--------|
| `authentik/lib/models.py` | Adds `WildcardDomainlessURLValidator` — accepts `*` in URLs |
| `authentik/providers/saml/models.py` | Uses the new validator on the `acs_url` field |
| `authentik/providers/saml/processors/authn_request_parser.py` | Matches incoming `AssertionConsumerServiceURL` against the provider's ACS URL using glob (`fnmatch`) when it contains `*`; stores the resolved concrete URL on `AuthNRequest` |
| `authentik/providers/saml/processors/assertion.py` | Uses the resolved concrete URL (not the pattern) for `Recipient` and `Destination` in the SAML response |
| `authentik/providers/saml/views/flows.py` | Uses the resolved concrete URL for the POST/Redirect binding target |

### Example

Set `acs_url` on the SAML provider to:
```
https://*.example.com/saml/acs
```

Any incoming `AssertionConsumerServiceURL` matching that pattern (e.g. `https://app1.example.com/saml/acs`) will be accepted, and the SAML response will be sent to the concrete URL — not the wildcard pattern.

---

## Building

The patch is applied on top of the official `ghcr.io/goauthentik/server` image.
All commands are run from the **repository root**.

### Option 1 — docker compose (single platform, current host)

```bash
cd lifecycle/container
docker compose build
docker compose up -d
```

`compose.override.yml` is picked up automatically alongside `compose.yml`.

### Option 2 — docker buildx (multi-platform: arm64 + amd64)

Use this when you develop on an Apple Silicon Mac but deploy to a Linux (amd64) server.

```bash
# one-time: create a multi-platform builder if you don't have one
docker buildx create --use --name multiarch

docker buildx build --platform linux/amd64,linux/arm64 --file lifecycle/container/Dockerfile.patch --build-arg AUTHENTIK_TAG=2026.5.3 --tag boussafawalid/authentik-patched:2026.5.3 --push .
```

Then reference the pre-built image in `compose.override.yml`:

```yaml
services:
  server:
    image: boussafawalid/authentik-patched:2026.5.3
  worker:
    image: boussafawalid/authentik-patched:2026.5.3
```

---

## Upgrading authentik

1. Update `AUTHENTIK_TAG` in `Dockerfile.patch` and `compose.override.yml` to the new version.
2. Rebuild: `docker compose build` or re-run the `docker buildx build` command with the new tag.
3. Verify the changed files still apply cleanly (no upstream conflicts in the 5 patched files).
