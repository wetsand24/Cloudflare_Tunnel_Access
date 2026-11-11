# Cloudflare Tunnel + Access for Uptime Kuma (no open ports)

The files from the video: Uptime Kuma and cloudflared in one Docker Compose file, published through a Cloudflare Tunnel and locked with Cloudflare Access (email one-time PIN).

▶️ Video: https://youtu.be/-m-GpkTv3Lo

My other videos: [Jellyfin setup](https://youtu.be/hurqeeMF53U) (to watch from outside, it uses Tailscale instead) · [Nginx Proxy Manager + Tailscale](https://youtu.be/U9tIwCmWctc) (names and HTTPS for your apps at home)

## What you need
- A domain with its DNS on Cloudflare. If you don't have a domain, you can buy one on Cloudflare.
  If your domain is at another registrar (GoDaddy, Namecheap...), keep it there: add it to Cloudflare and change its
  nameservers at the registrar to the two Cloudflare gives you (turn DNSSEC off at the registrar first if it's on).
  Keeping DNS at the registrar (partial/CNAME setup) only works on Cloudflare's Business and Enterprise plans.
- A free Cloudflare account (Zero Trust Free plan for Access)
- A Linux machine with Docker and Docker Compose (I used Debian 13 with Docker 29)

## No Docker yet? Install it (Debian 12/13)
From Docker's official guide: https://docs.docker.com/engine/install/debian/
```bash
# Docker's apt repository
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update

# Docker Engine + the Compose plugin
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# check
sudo docker run hello-world
docker compose version
```
To run `docker` without `sudo`, see https://docs.docker.com/engine/install/linux-postinstall/
Other distros (Ubuntu, Fedora...): https://docs.docker.com/engine/install/

## Steps
1. `mkdir cloudflare-demo && cd cloudflare-demo`, put `docker-compose.yml` from this repo in it.
   No `ports:` line: nothing is published on the host, only the tunnel talks to Uptime Kuma.
2. Cloudflare dashboard → Networking → Tunnels → Create Tunnel → name it → Docker. Copy the command.
3. Create `.env` next to the compose file with only the token (everything after `--token`):
   ```
   TUNNEL_TOKEN=eyJ...
   ```
   Treat the token like a password. If it leaks, rotate it on the tunnel's page.
4. `docker compose up -d`, then `docker compose logs cloudflared | grep Registered` shows four connections.
5. Tunnel → Add route → Published application:
   - Subdomain: `status`, Domain: your domain
   - Service URL: `http://uptime-kuma:3001` (the service name from the compose file, not `localhost`: cloudflared runs in its own container)
6. Open the address and finish Uptime Kuma's setup (SQLite, then your admin account) right away.
7. Turn on Zero Trust (first time only), see the section below.
8. Zero Trust → Access controls → Applications → Add an application → Self-hosted:
   - Same address as the route
   - Policy: Allow, Include: Emails, your address (any email works)
   - Login method: One-time PIN (accept all, with no other identity provider set up)

## Turn on Zero Trust (first time only)
Access is part of Cloudflare Zero Trust. The first time you open it, Cloudflare asks you to finish your account setup:
1. Cloudflare dashboard → **Zero Trust**.
2. Pick a **team name**. Your Access login page will be `<team-name>.cloudflareaccess.com`.
3. **Choose a plan** → **Free** ($0 / seat / month, 50 seat limit).
4. Checkout ("Protect up to 50 users at no cost", due today $0/month) still asks for a payment method. On the Free plan you are not charged ([Cloudflare's setup guide](https://developers.cloudflare.com/cloudflare-one/setup/)).

After that, Zero Trust → Access controls → Applications works as in step 8.

## More services
Same tunnel: Add route again, one hostname each. It can point at another machine on your network, for example `https://192.168.1.10:8006` for Proxmox (Additional application settings → TLS → turn on Disable TLS certificate verification, because Proxmox uses its own certificate). Every new hostname needs its own Access application, or one application for `*.yourdomain`, otherwise it is open to anyone.

## Good to know
- Your traffic goes through Cloudflare (one connection phone → Cloudflare, another Cloudflare → your server).
- Free plan: don't stream video (Jellyfin, Plex) through it; Cloudflare's terms ask for a paid service for video and large files.
- Keep the app's own login on. Access is an extra lock, not a replacement.

## Links
- Create a tunnel: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/
- Self-hosted Access app: https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/
- One-time PIN: https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/
- Uptime Kuma: https://github.com/louislam/uptime-kuma
