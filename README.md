# LiroxNotes container image

`ghcr.io/yoramdelangen/lirox-notes` runs the [LiroxNotes](https://github.com/yoramdelangen/lirox-notes)
server: a keyboard-first notes workbench whose notes live in a Git repository.

- Multi-arch: `linux/amd64` and `linux/arm64`.
- Built on `gcr.io/distroless/static:nonroot`. The image holds one static
  server binary and the web assets. It has no shell and no `git` binary;
  Git runs in-process through libgit2.
- Runs as uid/gid `65532` and listens on port `8080`.
- Every release ships with build provenance and an SBOM attestation.

> This repository only holds the package README. Source code, issues and
> releases are in [yoramdelangen/lirox-notes](https://github.com/yoramdelangen/lirox-notes).

## Tags

| Tag | Points to |
|---|---|
| `latest` | The newest stable release. |
| `X.Y` | The newest patch release of `X.Y`. |
| `X.Y.Z` | One exact release. Pin this one in production. |

Pre-releases only get the `X.Y.Z-…` tag.

## Quick start

```bash
docker run -d --name liroxnotes -p 8080:8080 \
  -v liroxnotes-config:/config \
  -v liroxnotes-data:/data \
  ghcr.io/yoramdelangen/lirox-notes:latest
```

Open <http://localhost:8080> to go through onboarding: register a passkey,
then create or connect the Git repository for your notes. Podman uses the same
command.

## Docker Compose

```yaml
services:
  liroxnotes:
    image: ghcr.io/yoramdelangen/lirox-notes:latest
    ports:
      - "8080:8080"
    environment:
      LIROX_RP_ORIGIN: https://notes.example.com
      LIROX_GIT_SSH_KEY: /run/secrets/git_key
      LIROX_GIT_KNOWN_HOSTS: /run/secrets/known_hosts
    volumes:
      - config:/config
      - data:/data
    secrets:
      - git_key
      - known_hosts
    restart: unless-stopped

volumes:
  config:
  data:

secrets:
  git_key:
    file: ./secrets/git_key
  known_hosts:
    file: ./secrets/known_hosts
```

Every variable is optional. Without them, onboarding asks for the domain and
you can only use local or credential-free remotes.

## Domains and HTTPS

Passkeys are bound to a domain. The first onboarding step asks where you open
LiroxNotes, prefills the address in your browser (port included), and stores
the answer in `/config`. Your passkey then only works on those domains.

- Passkeys need HTTPS. The only exception is `http://localhost`.
- Behind a reverse proxy or tunnel, do onboarding through the public
  `https://` URL, not the container's internal address.
- Any extra domain must use the first domain's host (on another port) or one
  of its subdomains.
- To set the domains from outside, set `LIROX_RP_ORIGIN` (and optionally
  `LIROX_RP_ID`). Onboarding then skips the domain step.

## Volumes

| Path | Contents |
|---|---|
| `/config` | App config, domains, sessions, passkeys, keymaps and editor options. |
| `/data` | The cloned note repositories (`/data/workspace`). |

Both are owned by uid/gid `65532`. Named volumes work as-is. Before you
bind-mount host directories, change their owner:

```bash
mkdir -p config data && sudo chown -R 65532:65532 config data
```

Back up both volumes. `/data` holds your notes and `/config` holds the passkeys
you sign in with.

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `LIROX_RP_ORIGIN` | — | Sets the onboarding domains: allowed origin(s), comma-separated, exact including port. |
| `LIROX_RP_ID` | first origin's host | Sets the passkey domain (RP ID). |
| `LIROX_SESSION_TTL_SECONDS` | 30 days | Session lifetime. |
| `IP` / `PORT` | `0.0.0.0` / `8080` | Listen address. |
| `LIROX_CONFIG_FILE` | `/config/config` | Application config (workspaces). |
| `LIROX_APP_CONFIG` | `/config/config.toml` | Keymaps and editor options. |
| `LIROX_WORKSPACE_DIR` | `/data/workspace` | Where repositories are cloned. |
| `LIROX_GIT_SSH_KEY` | — | Private key file for SSH remotes (no passphrase). |
| `LIROX_GIT_KNOWN_HOSTS` | `~/.ssh/known_hosts` | Pinned SSH host keys. Unknown hosts are refused. |
| `LIROX_GIT_TOKEN` | — | Token for HTTPS remotes. |
| `LIROX_GIT_USERNAME` | `x-access-token` | Username sent with the token. |
| `LIROX_GIT_TOKEN_HOSTS` | `github.com` | Hosts (comma-separated) that may receive the token. |

## Git credentials

Give the container its own credentials, limited to the notes repository.

**SSH deploy key (recommended)**

```bash
mkdir -p secrets
ssh-keygen -t ed25519 -f secrets/git_key -N ""
ssh-keyscan github.com > secrets/known_hosts
sudo chown 65532:65532 secrets/git_key && chmod 600 secrets/git_key
```

1. Add `secrets/git_key.pub` as a deploy key *with write access* on the notes
   repository.
2. Check the fingerprints in `secrets/known_hosts` against the ones GitHub
   publishes. You can also copy the keys from `https://api.github.com/meta`
   (`ssh_keys`). Keep every key type, because the server can negotiate any of
   them.
3. Mount both files and point `LIROX_GIT_SSH_KEY` and `LIROX_GIT_KNOWN_HOSTS`
   at them, as in the Compose example. With `docker run`:

   ```bash
   -v ./secrets/git_key:/run/secrets/git_key:ro \
   -v ./secrets/known_hosts:/run/secrets/known_hosts:ro \
   -e LIROX_GIT_SSH_KEY=/run/secrets/git_key \
   -e LIROX_GIT_KNOWN_HOSTS=/run/secrets/known_hosts
   ```

**HTTPS token**

Create a fine-grained token for that one repository with *Contents: read and
write* and an expiry date, and pass it as `LIROX_GIT_TOKEN`. The token is only
sent to hosts listed in `LIROX_GIT_TOKEN_HOSTS`.

The image does not use Git credential helpers or `~/.ssh/config`.

## Upgrading

```bash
docker pull ghcr.io/yoramdelangen/lirox-notes:latest
docker rm -f liroxnotes
# re-run the `docker run` command above. With Compose: docker compose up -d
```

Your config and notes stay in the volumes.

## Verifying an image

```bash
gh attestation verify oci://ghcr.io/yoramdelangen/lirox-notes:<version> \
  -R yoramdelangen/lirox-notes
```

## Building it yourself

From a checkout of [the source repository](https://github.com/yoramdelangen/lirox-notes):

```bash
just image        # builds lirox-notes:dev
just image-run    # runs it on http://localhost:8080
```
