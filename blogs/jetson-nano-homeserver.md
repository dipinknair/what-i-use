---
layout: default
title: "How I Turned My NVIDIA Jetson Nano Into a Home Server"
---

# How I Turned My NVIDIA Jetson Nano Into a Home Server: Pi-hole, Mini NAS & VPN


> A step-by-step guide to setting up Pi-hole for network-wide ad blocking, a self-hosted Nextcloud instance for private cloud file access, and Tailscale VPN with Magic DNS for seamless remote access — all on a Jetson Nano.

---

## What I Built

My NVIDIA Jetson Nano sits quietly on my desk, drawing only ~5W of power, doing three very useful things around the clock:

- 🚫 **Blocking ads** for every device on my home network using Pi-hole
- 💾 **Serving files** from a 500GB USB SSD via self-hosted Nextcloud, accessible at a friendly Tailscale Magic DNS URL
- 🌍 **Allowing remote access** from anywhere in the world via Tailscale VPN

Here's exactly how I set it all up.

---

## Hardware

- **NVIDIA Jetson Nano 4GB**
- **MicroSD card** (15GB — boot drive)
- **500GB USB SSD** (for Pi-hole data, Docker, and file storage)
- **Ethernet cable** (wired connection for reliability)

---

## Part 1: Formatting and Mounting the SSD

The Jetson Nano boots from a MicroSD card which only has 15GB of space. I connected a 500GB USB SSD to offload storage.

### Step 1: Identify the SSD

```bash
lsblk
```

My SSD showed up as `/dev/sda`. The boot MicroSD was `/dev/mmcblk0`. Make sure you target the right drive!

### Step 2: Unmount if auto-mounted

Ubuntu's desktop automatically mounts USB drives. Unmount it first:

```bash
sudo umount /dev/sda1
```

### Step 3: Partition and format

```bash
# Create a GPT partition table with a single partition
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%

# Format to ext4
sudo mkfs.ext4 -F /dev/sda1
```

### Step 4: Mount and set permissions

```bash
sudo mkdir -p /mnt/ssd
sudo mount /dev/sda1 /mnt/ssd
sudo chown -R $USER:$USER /mnt/ssd
```

### Step 5: Auto-mount on boot

Get the UUID:

```bash
sudo blkid /dev/sda1
```

Add it to `/etc/fstab`:

```bash
echo "UUID=YOUR-UUID-HERE /mnt/ssd ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

The `nofail` option is important — it ensures your Nano still boots even if the SSD is unplugged.

---

## Part 2: Installing Docker (with Storage on the SSD)

I use Docker to run Pi-hole as a container. I configured Docker to store all data on the SSD so the MicroSD doesn't fill up.

### Install Docker

```bash
sudo apt-get update
sudo apt-get install -y docker.io
```

### Point Docker storage to the SSD

```bash
sudo systemctl stop docker

sudo tee /etc/docker/daemon.json <<EOF
{
  "data-root": "/mnt/ssd/docker"
}
EOF

sudo systemctl daemon-reload
sudo systemctl start docker
sudo systemctl enable docker
```

### Allow Docker without sudo

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Test Docker

> **Note for Jetson Nano users:** The Jetson Nano runs an older kernel (4.9.x) that has a known conflict with Docker's default seccomp profile. Always add `--security-opt seccomp=unconfined` when running containers.

```bash
docker run --rm --security-opt seccomp=unconfined hello-world
```

You should see `Hello from Docker!` ✅

---

## Part 3: Installing Pi-hole (Network-Wide Ad Blocker)

Pi-hole is a DNS-level ad blocker. Instead of blocking ads in a browser, it blocks them at the DNS level — meaning **every device on your network** gets ad blocking automatically, including phones, smart TVs, and gaming consoles.

### Fix port 53 conflict

Ubuntu uses `systemd-resolved` which occupies port 53. Pi-hole needs that port for DNS. Disable the stub listener:

```bash
echo "DNSStubListener=no" | sudo tee -a /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

### Run Pi-hole in Docker

```bash
docker run -d \
  --name pihole \
  --security-opt seccomp=unconfined \
  --network host \
  --restart unless-stopped \
  -e TZ="America/Chicago" \
  -e WEBPASSWORD="your_password_here" \
  -v /mnt/ssd/pihole/etc-pihole:/etc/pihole \
  -v /mnt/ssd/pihole/etc-dnsmasq.d:/etc/dnsmasq.d \
  pihole/pihole:latest
```

> Replace `America/Chicago` with your timezone and set a strong password for the web admin panel.

### Verify Pi-hole is running

```bash
docker ps
```

Wait about 60 seconds for the status to change from `health: starting` to `healthy`.

### Access the admin panel

Open a browser on any device on your network:

```
http://192.168.1.200/admin
```

You'll see the Pi-hole dashboard with real-time DNS query statistics and blocked domains.

### Point your devices to Pi-hole

Since my Spectrum router is locked down and doesn't allow changing DNS settings, I manually set the DNS on each device to `192.168.1.200`.

If your router allows DNS changes, set:
- **Primary DNS:** `192.168.1.200` (Pi-hole)
- **Secondary DNS:** `8.8.8.8` (Google — fallback if Nano is offline)

---

## Part 4: Self-Hosted Cloud Storage with Nextcloud

Nextcloud gives you a private Google Drive — a full web UI for files, with desktop and mobile sync clients, all running on your own hardware. Combined with Tailscale Magic DNS, you get a beautiful `https://your-machine-name.tail-id.ts.net` URL that works from anywhere.

### Run Nextcloud in Docker

```bash
sudo mkdir -p /mnt/ssd/nextcloud

docker run -d \
  --name nextcloud_app_1 \
  --security-opt seccomp=unconfined \
  --restart unless-stopped \
  -p 8080:80 \
  -v /mnt/ssd/nextcloud:/var/www/html \
  nextcloud:latest
```

> **Note:** Port 8080 on the Nano maps to port 80 inside the container. Nextcloud data (config, files, database) all lives on the SSD under `/mnt/ssd/nextcloud`.

### First-run setup

Open a browser on your local network:

```
http://192.168.1.200:8080
```

Nextcloud will prompt you to:
1. Create an **admin account** (pick a strong password)
2. Choose **SQLite** for a simple single-user setup, or MariaDB for multi-user
3. Click **Finish setup** — it takes about 30 seconds

### Migrate files from the old Samba share

If you had files in `/mnt/ssd/share`, copy them directly into Nextcloud's data folder:

```bash
# Copy all files (preserves permissions and timestamps)
sudo cp -av /mnt/ssd/share/* /mnt/ssd/nextcloud/data/<your-username>/files/

# Fix ownership — Nextcloud runs as www-data (UID/GID 33) inside the container
sudo chown -R 33:33 /mnt/ssd/nextcloud/data/<your-username>/files/

# Tell Nextcloud to scan and index the new files
docker exec -it nextcloud_app_1 su -s /bin/bash www-data -c \
  "php occ files:scan --path=/<your-username>/files"
```

Refresh the Nextcloud web UI — your files will appear. Verify a few different types (docs, images, etc.) before deleting the original `/mnt/ssd/share`.

### Enable Tailscale Magic DNS HTTPS

Tailscale's Magic DNS gives every device on your tailnet a stable hostname like `jetson.xxxxxxxxxxxx.ts.net`. Enable HTTPS so your Nextcloud URL is properly secured:

```bash
# Enable HTTPS on the Tailscale node (one-time)
sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-routes
```

In the [Tailscale admin console](https://login.tailscale.com/admin/dns), enable **HTTPS Certificates** under the DNS tab.

Then add your Magic DNS hostname to Nextcloud's trusted domains:

```bash
docker exec -it nextcloud_app_1 su -s /bin/bash www-data -c \
  "php occ config:system:set trusted_domains 1 --value='jetson.xxxxxxxxxxxx.ts.net'"
```

> Replace `jetson.xxxxxxxxxxxx.ts.net` with your actual Magic DNS hostname (find it with `tailscale status`).

Now access your Nextcloud from any device on your tailnet at:

```
https://jetson.xxxxxxxxxxxx.ts.net:8080
```

---

## Part 5: Remote Access with Tailscale VPN

Nextcloud and Pi-hole work great on the local network. But what about accessing your files when you're away from home? This is where **Tailscale** comes in — and with Magic DNS, you don't even need to remember an IP address.

Tailscale is a zero-config VPN built on WireGuard. It works without any port forwarding or router changes — perfect for locked-down ISP routers.

### Install Tailscale on the Nano

```bash
sudo apt-get install -y curl
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

It will print a URL — open it in your browser to authenticate. Sign up for a free Tailscale account (supports up to 100 devices for free).

### Get your Nano's Tailscale IP

```bash
tailscale ip
```

You'll get an IP like `100.x.x.x`.

### Install Tailscale on your Mac

Download from [tailscale.com/download](https://tailscale.com/download) and sign in with the same account.

### Access everything remotely

Once both devices are on Tailscale, from anywhere in the world you can:

| Service | URL |
|---------|-----|
| **Nextcloud Files** | `https://jetson.xxxxxxxxxxxx.ts.net:8080` |
| **Pi-hole Dashboard** | `http://jetson.xxxxxxxxxxxx.ts.net/admin` |
| **SSH** | `ssh your-username@jetson.xxxxxxxxxxxx.ts.net` |

> Replace `jetson.xxxxxxxxxxxx.ts.net` with your actual Magic DNS hostname from `tailscale status`.

> **Performance note:** Tailscale uses direct peer-to-peer connections (WireGuard encrypted) so overhead is minimal. Speed is limited mainly by your home internet upload speed.

---

## Final Setup Overview

```
NVIDIA Jetson Nano (192.168.1.200)
├── Boot Drive: 15GB MicroSD
├── Storage: 500GB USB SSD mounted at /mnt/ssd
│   ├── /mnt/ssd/docker      ← Docker images & containers
│   ├── /mnt/ssd/pihole      ← Pi-hole config & database
│   └── /mnt/ssd/nextcloud   ← Nextcloud data & config
├── Services
│   ├── Pi-hole (Docker)     ← DNS ad blocker, port 53
│   ├── Nextcloud (Docker)   ← Private cloud, port 8080
│   └── Tailscale            ← WireGuard VPN + Magic DNS
└── Access
    ├── Local:  http://192.168.1.200:8080
    └── Remote: https://jetson.xxxxxxxxxxxx.ts.net:8080 (via Tailscale Magic DNS)
```

---

## What's Next?

Here are some ideas to expand this setup:

- **Portainer** — Web UI to manage all Docker containers visually
- **Jellyfin** — Stream movies and music from the SSD to any device
- **Nextcloud Desktop Sync** — Install the Nextcloud client on your Mac for automatic two-way sync
- **Home Assistant** — Smart home automation hub
- **AI / Computer Vision** — Use the Jetson Nano's built-in 128-core NVIDIA GPU for local AI inference

---

## Conclusion

For about $100 (Nano + SSD), I have a low-power home server that:
- Blocks ads on every device without installing anything on them
- Serves files through a private Nextcloud instance, accessible at a real HTTPS URL via Tailscale Magic DNS
- Is accessible from anywhere via encrypted VPN — no port forwarding, no dynamic DNS hacks
- Runs 24/7 on ~5W of power

The Jetson Nano is an excellent platform for a home server, especially if you want to venture into AI and computer vision projects down the road. Happy tinkering! 🚀
