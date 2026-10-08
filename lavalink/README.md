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

### Connection details

There is no live node address or usable password yet. `music.example.com` and the password in `.env.example` are placeholders, not a public Lavalink service. Complete the steps below to create your own.

| Bot setting | Value after deployment |
| --- | --- |
| Node name / identifier | `AWS-Lavalink` (a label you choose in your bot) |
| Host | Your DNS hostname, e.g. `music.yourdomain.com`, without `https://` |
| Port | `443` |
| Secure / TLS | `true` |
| Password | The value you generate for `LAVALINK_SERVER_PASSWORD` in `.env` |
| Base URL | `https://music.yourdomain.com` |
| Lavalink v4 WebSocket URL | `wss://music.yourdomain.com/v4/websocket` |

Lavalink listens on `2333` inside Docker. Do not use `2333` in an external bot connection with this configuration. The node name is not a username, and the EC2 name does not set the bot's node name.

You need an AWS account, an SSH key pair, and a domain/subdomain whose DNS you can edit. This guide uses a domain for Caddy's automatic HTTPS. An IP address alone is not a drop-in replacement for this setup.

### 1. Create the instance

Use an Ubuntu 24.04 LTS EC2 instance, preferably x86-64, with at least 2 vCPUs and 4 GiB RAM for a first deployment. Assign an Elastic IP and create a DNS `A` record for a hostname such as `music.example.com` pointing to that IP.

1. Open the [AWS EC2 console](https://console.aws.amazon.com/ec2/) and choose a region near your bot.
2. Click **Launch instance**. Name it `AWS-Lavalink`.
3. Choose **Ubuntu Server 24.04 LTS**, architecture **64-bit (x86)**.
4. Select a 2-vCPU, 4-GiB instance such as `t3.medium`. This is a starting point, not a guarantee of free-tier eligibility or capacity.
5. Create/select a key pair. Download its `.pem` file and keep it private.
6. Use a public subnet with internet access, enable a public IP, and choose approximately 20 GiB of gp3 storage.
7. Create a security group with the rules below, then launch. Wait for the instance status checks to pass.
8. Under **Network & Security → Elastic IPs**, allocate an address and associate it with this instance.
9. At your DNS provider (or Route 53), create an `A` record for your chosen subdomain pointing to the Elastic IP. If your DNS provider offers an HTTP proxy, use DNS-only initially. Do not add an `AAAA` record unless you also configured IPv6.

AWS charges can include instance usage, storage, public IPv4/Elastic IPs, and bandwidth. Check your account's pricing and credits before launching. Stopping an instance does not remove all charges; release unused resources when you no longer need them.

In the EC2 security group, allow:

- TCP `80` and `443` from the internet, for Caddy's HTTP redirect and HTTPS endpoint.
- TCP `22` only from your own IP address, for SSH.

Do **not** add an inbound rule for TCP `2333`. Lavalink is only reachable from the private Docker network; your bot connects through the HTTPS hostname on port `443`.

Keep outbound internet access enabled for image/plugin downloads, source requests, and certificate issuance.

### 2. Install Docker Compose

On your computer, replace `YOUR_KEY.pem` and `YOUR_ELASTIC_IP` in these commands:

```bash
chmod 400 YOUR_KEY.pem
ssh -i YOUR_KEY.pem ubuntu@YOUR_ELASTIC_IP
```

On Windows, use PowerShell/OpenSSH and restrict the key file permissions instead of using `chmod`. You can also follow **EC2 → Instances → Connect → SSH client** for the exact command.

In the SSH session on Ubuntu, install Docker and the tools used below:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 openssl curl dnsutils
sudo systemctl enable --now docker
sudo docker compose version
```

This guide uses `sudo docker` so you do not need to change Docker group membership.

### 3. Upload the Lavalink files

Download/clone your repository on your computer. From the repository root, run this in a **local terminal**, not the SSH session:

```bash
scp -i YOUR_KEY.pem -r lavalink ubuntu@YOUR_ELASTIC_IP:~/
```

Alternatively, clone your repository on EC2 and copy its `lavalink/` directory to `~/lavalink`. Only this directory is needed; you do not need to run the TypeScript workspace, install Node.js, or install Java separately.

Back in your **EC2 SSH session**:

```bash
cd ~/lavalink
ls -a
cp .env.example .env
nano .env
```

The directory must contain `compose.yaml`, `application.yml`, `Caddyfile`, and `.env.example`. Set `LAVALINK_DOMAIN` in `.env` to your real hostname, without a URL scheme, port, or path.

### 4. Set a private password

Run this command **on your own EC2 server**, not in a shared chat:

```bash
openssl rand -hex 32
```

Copy the generated value, run `nano .env` again, and replace the placeholder after `LAVALINK_SERVER_PASSWORD=` with it. Save with **Ctrl+O**, Enter, then exit with **Ctrl+X**. Do not leave the sample password in place.

Store the same value privately in your bot's secret configuration. Never commit `.env`, publish the password, or paste it into chat. There is no default password intended for use with this project.

Restrict access to the environment file and prepare the plugin cache directory:

```bash
chmod 600 .env
mkdir -p plugins
sudo chown -R 322:322 plugins
```

### 5. Start the node

Check your real hostname resolves to your Elastic IP:

```bash
dig +short music.yourdomain.com A
```

From `~/lavalink`, validate and start both containers:

```bash
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose logs -f lavalink
```

Use **Ctrl+C** to exit log viewing; the containers keep running.

Caddy will request and renew the HTTPS certificate automatically. The initial certificate can take a short time to issue. To check service status:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 caddy
```

The node is reachable at `https://<your-domain>` on port `443`; the bot should use that hostname, enable TLS, and use the password from `.env`. The exact node options vary by the bot's Lavalink client library.

To verify the authenticated Lavalink API after Caddy has obtained its certificate:

```bash
set -a
. ./.env
set +a
curl -fsS -H "Authorization: $LAVALINK_SERVER_PASSWORD" "https://$LAVALINK_DOMAIN/v4/info"
```

Successful verification returns JSON describing Lavalink and its plugins. Do not use `curl -k` to ignore a certificate failure; fix DNS or certificate issuance instead.

### 6. Connect your Discord bot

Use the connection details above in the bot's existing Lavalink client. A typical configuration has these fields; exact names depend on your client library:

```js
{
  name: "AWS-Lavalink",
  host: process.env.LAVALINK_HOST,
  port: 443,
  password: process.env.LAVALINK_PASSWORD,
  secure: true
}
```

`LAVALINK_HOST` is your real DNS hostname. `LAVALINK_PASSWORD` must equal the server's `LAVALINK_SERVER_PASSWORD`; these two variable names are used in different processes.

Some clients use `identifier` instead of `name`, `authorization` instead of `password`, or a URL field instead of host/port. This snippet is a mapping example, not a standalone bot. Use a Lavalink v4-compatible client; the client supplies the WebSocket headers, including authorization and the Discord bot's user ID.

Start your bot and check its node-connected log, then play a test track. YouTube may block cloud/datacenter IPs even when the node is correctly configured; a successful `/v4/info` response does not guarantee every source will play.

### 7. Update and operate the service

The Lavalink image follows the stable v4 line; the YouTube plugin is intentionally pinned. Review upstream release notes before updating either one. To pull the current image and restart:

```bash
sudo docker compose pull
sudo docker compose up -d
```

Useful diagnostics:

```bash
sudo docker compose logs --tail=100 lavalink
sudo docker compose logs --tail=100 caddy
```

Both containers restart after an EC2 reboot while Docker is enabled. To stop the service without removing persisted Caddy certificates, run `sudo docker compose down` from `~/lavalink`.

### Troubleshooting

- **Timeout / connection refused:** check the instance is running, the Elastic IP and DNS match, and inbound TCP 80/443 are allowed. Check `sudo docker compose ps`.
- **Certificate error:** check DNS and Caddy logs. Ports 80/443 must reach this server and not be occupied by another service.
- **HTTP 401:** the bot/request password does not match the server password. After changing `.env`, run `sudo docker compose up -d` and update the bot's private configuration too.
- **Plugin permission error:** from `~/lavalink`, run `sudo chown -R 322:322 plugins`, then `sudo docker compose restart lavalink`.
- **Node connects but tracks fail:** inspect Lavalink logs and test another source. Source availability, YouTube restrictions, and bot/client errors are separate from network setup.
- **Service is killed or unstable:** inspect `sudo docker stats --no-stream` and `free -h`; increase instance capacity if needed. Very small instances are unsuitable for the configured memory limits.

## References

- [Lavalink releases](https://github.com/lavalink-devs/Lavalink/releases)
- [Official Lavalink Docker guide](https://lavalink.dev/getting-started/docker.html)
- [Official YouTube source plugin and configuration](https://github.com/lavalink-devs/youtube-source)
- [YouTube source plugin releases](https://github.com/lavalink-devs/youtube-source/releases)
- [AWS EC2 launch and connection guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
