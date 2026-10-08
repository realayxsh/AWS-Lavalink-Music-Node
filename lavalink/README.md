# AWS Lavalink node

A self-hosted Lavalink node for a Discord music bot. It uses the official Lavalink v4 Docker image, Lavalink's maintained YouTube source plugin, the built-in audio filters, and Caddy for automatic HTTPS.

At preparation time, Lavalink `4.2.2` and `youtube-source` plugin `1.18.2` were the latest stable releases found in their upstream release feeds. The official `4-alpine` image tag follows the current stable Lavalink v4 line; the YouTube plugin version is pinned in `application.yml`.

## What this provides

- Lavalink v4 audio node with YouTube search/track loading through the official `youtube-source` plugin.
- Built-in support enabled for Bandcamp, SoundCloud, Twitch, Vimeo, and NicoNico.
- Lavalink's equalizer, volume, timescale, karaoke, tremolo, vibrato, distortion, rotation, channel-mix, and low-pass filters enabled.
- Opus encoding quality 10 and high resampling quality. These settings use more CPU; start with a 2-vCPU, 4-GiB EC2 instance and scale based on active listeners.
- HTTPS/WSS through Caddy with automatically managed certificates.
- Lavalink's raw port `2333` is not published to the internet. Only the Caddy web ports are exposed.
- A long random password is supplied through `.env`; no password is committed in the configuration.

This is an audio node, not a complete Discord bot. The bot still needs Lavalink client code, queue controls, and commands that call the filters. A Lavalink node can improve source support and playback controls, but it cannot make the bot's code or feature set “premium” by itself. Spotify links are not enabled by this setup; they require an additional source/metadata integration and may need provider credentials.

## Deploy to AWS EC2

### 1. Create the instance

Use an Ubuntu 24.04 LTS EC2 instance, preferably x86-64, with at least 2 vCPUs and 4 GiB RAM for a first deployment. Assign an Elastic IP and create a DNS `A` record for a hostname such as `music.example.com` pointing to that IP.

In the EC2 security group, allow:

- TCP `80` and `443` from the internet, for Caddy's HTTP redirect and HTTPS endpoint.
- TCP `22` only from your own IP address, for SSH.

Do **not** add an inbound rule for TCP `2333`. Lavalink is only reachable from the private Docker network; your bot connects through the HTTPS hostname on port `443`.

### 2. Install Docker Compose

Connect to the EC2 instance and install Docker:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
docker compose version
```

Copy the `lavalink/` folder from this project to the instance. For example, after placing it at `/opt/lavalink`:

```bash
cd /opt/lavalink
cp .env.example .env
nano .env
```

Set `LAVALINK_DOMAIN` to the DNS name you configured, and replace the example password with a unique random value:

```bash
openssl rand -hex 32
```

Paste the generated value into `LAVALINK_SERVER_PASSWORD` in `.env`. Use the same value in your bot's Lavalink-node configuration. Then restrict access to the environment file and prepare the plugin cache directory:

```bash
chmod 600 .env
mkdir -p plugins
sudo chown -R 322:322 plugins
```

### 3. Start the node

Make sure the DNS record resolves to the EC2 Elastic IP, then start both containers:

```bash
docker compose up -d
docker compose logs -f lavalink
```

Caddy will request and renew the HTTPS certificate automatically. The initial certificate can take a short time to issue. To check service status:

```bash
docker compose ps
```

The node is reachable at `https://<your-domain>` on port `443`; the bot should use that hostname, enable TLS, and use the password from `.env`. The exact node options vary by the bot's Lavalink client library.

To verify the authenticated Lavalink API after Caddy has obtained its certificate:

```bash
set -a
. ./.env
set +a
curl -fsS -H "Authorization: $LAVALINK_SERVER_PASSWORD" "https://$LAVALINK_DOMAIN/v4/info"
```

### 4. Update the service

The Lavalink image follows the stable v4 line; the YouTube plugin is intentionally pinned. Review upstream release notes before updating either one. To pull the current image and restart:

```bash
docker compose pull
docker compose up -d
```

Useful diagnostics:

```bash
docker compose logs --tail=100 lavalink
docker compose logs --tail=100 caddy
```

## References

- [Lavalink releases](https://github.com/lavalink-devs/Lavalink/releases)
- [Official Lavalink Docker guide](https://lavalink.dev/getting-started/docker.html)
- [Official YouTube source plugin and configuration](https://github.com/lavalink-devs/youtube-source)
- [YouTube source plugin releases](https://github.com/lavalink-devs/youtube-source/releases)
