# tsdproxy for self-hosting

I used this to expose services running on Dokploy through Tailscale.

Rough instructions for setting this up below:

**Tsdproxy + Dokploy setup (generic)**

**1. Tailnet policy file → add a tag for tsdproxy-created devices.** Merge in:

```json
"tagOwners": {
  "tag:tsdproxy": []
}
```

**2. Tailscale admin → Settings → Keys → Generate auth key.**
- Reusable: yes
- Ephemeral: no
- Tags: `tag:tsdproxy`
- Expiry: 90 days (max via UI)

Copy the key.

**3. Create a repo for the tsdproxy compose** (e.g. `your-org/tsdproxy`) with `docker-compose.yml`:

```yaml
services:
  tsdproxy:
    image: almeidapaulopt/tsdproxy:2.1.0
    restart: unless-stopped
    environment:
      - TSDPROXY_AUTHKEY
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - tsdproxy_data:/data
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  tsdproxy_data:
```

Commit and push.

**4. In Dokploy: create a new compose service pointing at the repo.**
- Env var: `TSDPROXY_AUTHKEY=tskey-auth-xxxxx`
- Deploy

**5. Add labels to each app's compose and redeploy.** For each service you want exposed on the tailnet:

```yaml
    labels:
      tsdproxy.enable: "true"
      tsdproxy.name: "yourservice"
      tsdproxy.port.1: "443/https:<container-port>/http"
```

Where `<container-port>` is whatever port the app listens on inside the container.

**6. Tailscale admin → Machines.** For each new tsdproxy-spawned device (`yourservice`, etc.): Disable key expiry.

**7. If you previously had `tailscale serve` running on the host, clear it:**

```sh
tailscale serve reset
```

**8. Verify from another tailnet device:**

```sh
curl -sI https://yourservice.<your-tailnet>.ts.net
```

**9. Calendar reminder: rotate `TSDPROXY_AUTHKEY` every ~80 days.**
