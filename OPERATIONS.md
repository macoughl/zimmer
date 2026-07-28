# Operations: Mike's Zimmer instance

This fork of [tadasant/zimmer](https://github.com/tadasant/zimmer) runs Mike Coughlin's
orchestrator instance at `zimmer.transparentmetrics.com` (staging:
`zimmer-staging.transparentmetrics.com`). This document explains the moving pieces in
plain terms. Upstream's full docs live at
[docs.zimmer.tadasant.com](https://docs.zimmer.tadasant.com/).

## The stack, in one paragraph

A single DigitalOcean droplet (in the "Hermes" team) runs the Rails app and its worker
as Docker containers, deployed by Kamal from GitHub Actions. Terraform creates and
manages the droplet itself. The droplet is reachable only over a Tailscale private
network; it has no public web exposure. DNS and TLS certificates for the custom domain
come from Cloudflare (the transparentmetrics.com zone) via a scheduled cert workflow.
Container images live in GitHub Container Registry under `ghcr.io/macoughl`.

## Who does what

| Piece | Job | Where it runs |
|---|---|---|
| Terraform | Creates/updates the droplet, firewall, and networking from code. Its memory of what exists (the "state") lives in the `tm-zimmer-tfstate` Spaces bucket | Inside the deploy workflow |
| Kamal | Deploys the app: SSHes to the droplet, swaps in the new container only after it passes health checks, one-command rollback. Also runs Postgres and Redis containers on staging | Inside the deploy workflow |
| Tailscale | The private network. Only enrolled devices (Mike's machines, the droplet, CI during a deploy) can reach the app at all | Everywhere, as a background client |
| Cloudflare | Hosts DNS for transparentmetrics.com; the cert workflow uses its API to prove domain ownership to Let's Encrypt and to keep the A record pointed at the droplet's tailnet IP | Cloud service |
| GHCR | Stores built container images. CI pushes with its own token; the droplet pulls with a read-only PAT | Cloud service |

## Why Tailscale is load-bearing (read this before "simplifying" it away)

Zimmer has no login screen. Anyone who can reach the web UI can read the operator's
Anthropic and GitHub tokens in plaintext and can spawn agent sessions that hold real
production credentials. The security model, by upstream design, is: the app is only
reachable over a private network. Tailscale is that private network, an encrypted
peer-to-peer mesh limited to devices enrolled in this tailnet
(mike@transparentmetrics.com).

The alternatives are all worse: exposing the app publicly means bolting an auth system
onto upstream (a permanent fork and a permanent security burden); firewall IP
allowlists break with dynamic home IPs and leave CI stranded; self-managed WireGuard is
the same idea as Tailscale with more maintenance. Tailscale has no server component to
maintain and the free tier covers this use. Never expose the droplet's web port
publicly.

Tailnet identity: the tailnet belongs to the Transparent-Metrics GitHub organization,
logged in via GitHub SSO as macoughl (Tailscale requires an identity provider, not an
email signup). Membership therefore follows TM org membership on GitHub.

## Break-glass SSH (the `admin_ssh_pubkeys` list)

Everyday SSH to the droplet uses Tailscale SSH (port 22 on the tailnet), where
Tailscale's identity layer decides who gets in. The break-glass door is a second, plain
OpenSSH listener on port 2222, also tailnet-only, that instead accepts the public keys
listed in `infra/terraform/staging.tfvars.example` under `admin_ssh_pubkeys`, as root.
It exists for when Tailscale's identity/auth layer is the thing that is broken while the
network still works: `ssh -p 2222 root@<droplet tailnet IP>`.

Mike's laptop key (`~/.ssh/id_rsa` on Mike-SLS) is authorized there. Public keys are
safe to publish (they are the lock, not the key). If the laptop or key is ever lost,
remove the entry and redeploy with `recreate_droplet` to revoke it. If the entire
tailnet is unreachable, the true last resort is DigitalOcean's web recovery console
(droplet page -> Access -> Launch Recovery Console).

## Secrets inventory (GitHub Actions secrets on this repo)

Never commit secret values. All of these live in Settings -> Secrets and variables ->
Actions.

| Secret | What it is | Source |
|---|---|---|
| DIGITALOCEAN_ACCESS_TOKEN | Terraform's key to create/manage droplets | DO "Hermes" team -> API |
| SPACES_ACCESS_KEY_ID / SPACES_SECRET_ACCESS_KEY | Access to the tfstate bucket | DO Spaces keys |
| CLOUDFLARE_API_TOKEN | DNS edit rights on the transparentmetrics.com zone only | Cloudflare TM account |
| TAILSCALE_AUTH_KEY | Lets the new droplet join the tailnet | Tailscale admin console |
| TS_CI_AUTHKEY | Lets a CI job join the tailnet briefly to deploy | Tailscale admin console |
| TS_API_CLIENT_ID / TS_API_CLIENT_SECRET | Lets the deploy workflow clean up stale CI devices | Tailscale OAuth client |
| KAMAL_SSH_PUBKEY / KAMAL_SSH_KEY | Deploy keypair Kamal uses to SSH to the droplet | Generated locally |
| GHCR_PULL_TOKEN | Read-only token the droplet uses to pull images | GitHub (macoughl) PAT, scope read:packages |
| STAGING_SECRET_BASE / STAGING_API_KEYS / STAGING_DB_PASSWORD / STAGING_RAILS_MASTER_KEY | App secrets (Rails session signing, REST/MCP API keys, DB password, credentials key) | Generated locally |

Account worlds: DigitalOcean, Cloudflare, and Tailscale belong to the Transparent
Metrics world (mike@transparentmetrics.com). GitHub is the personal `macoughl` account,
which owns this fork and is the identity the instance's sessions use. PulseMCP-world
credentials (the ClawdBot Slack token, per-MCP-server credentials) enter later as app
runtime secrets, never as deploy secrets.

## Routine operations

- Deploy staging: Actions -> "deploy-staging" -> Run workflow. Rebuild the droplet from
  scratch only when bootstrap config changed: same workflow with `recreate_droplet`.
- Rollback: `bin/kamal rollback <version> -d staging` (versions: `bin/kamal audit`).
- After any droplet rebuild (not every deploy): exec into the container
  (`bin/kamal app exec -i --reuse -d staging bash`) and re-run `gh auth login` and
  `claude /login`. Sessions fail at the git-clone step until this is done.
- Certificates renew via the weekly "domain-cert-staging" workflow; it also re-points
  the domain's A record at the droplet's tailnet IP.
- Base image: monthly "build-base-image" workflow; run it manually after changing
  `Dockerfile.base`.
