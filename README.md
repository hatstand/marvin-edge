# marvin-edge

The Caddy that owns ports 80 and 443 on marvin and routes each public domain to its app's
compose stack:

| Domain | Upstream on the `edge` network | App |
| --- | --- | --- |
| metabeer.co.uk (and www) | `metabeer-web:8000` | [Metabeer/metabeer-v2](https://github.com/Metabeer/metabeer-v2) (`compose/edge.yaml`) |
| triptych.purplehatstands.com | `triptych-proxy:8000` | [hatstand/triptych](https://github.com/hatstand/triptych) (`compose/edge.yaml`) |

Adding a site takes one block in [`caddy/Caddyfile`](caddy/Caddyfile), plus a row here. The app
puts one service on the external `edge` network under an alias, and Caddy gets the site's
certificate itself once DNS points at marvin.

## Deploys

Every push to `main` runs `check` (`caddy validate` and `caddy fmt --diff` against the pinned
image), then `deploy`: it joins the tailnet as an ephemeral `tag:ci` node through Tailscale
workload identity federation and runs `tailscale ssh edge-deploy@marvin <sha>`. That user's
login shell is [`deploy/edge-deploy`](deploy/edge-deploy), which:

1. accepts only a commit on `main` and checks it out in `/srv/edge`;
2. validates the new Caddyfile with the pinned image, and if it fails, goes back to the old checkout;
3. if only the Caddyfile changed, runs `caddy reload`, which is graceful and drops no connections.
   Otherwise it recreates the container, leaving 80/443 down for a second or so;
4. curls every site over TLS on the LAN address and prints the status.

Pull requests get the same `check`, so a broken Caddyfile can't reach `main`.

Rolling back is a revert PR, or `tailscale ssh edge-deploy@marvin <older sha>` from a tailnet device.

## One-time setup

### Tailscale

Add `edge-deploy` to the `tag:ci` → `tag:marvin` SSH rule and its `sshTests` (see triptych's
`deploy/README.md`). Then Trust credentials → **Credential** → **OpenID Connect** → issuer
**GitHub**, with subject `repo:hatstand@314491/marvin-edge@1442986314:environment:marvin` (Tailscale
shows the subject it received if this doesn't match), scope **Auth Keys (write)** and tag `tag:ci`.

### GitHub

Environments → `marvin` → deployment branches: `main` only. Add the variables
`TS_OAUTH_CLIENT_ID` and `TS_AUDIENCE`.

### marvin

```bash
sudo install -m 0755 deploy/edge-deploy /usr/local/bin/edge-deploy
echo /usr/local/bin/edge-deploy | sudo tee -a /etc/shells
sudo useradd --system --create-home --shell /usr/local/bin/edge-deploy --groups docker edge-deploy
sudo git clone https://github.com/hatstand/marvin-edge /srv/edge
echo "PUBLIC_BIND=<marvin's LAN address>" | sudo tee /srv/edge/.env
sudo chown -R edge-deploy: /srv/edge
```

Reinstall the shell whenever `deploy/edge-deploy` changes, because the login shell is a copy of
the script, not the checkout.

The first start takes 80/443 over from triptych's `proxy`. triptych's `deploy/README.md`,
section "Behind the shared edge", has the order to do that in.
