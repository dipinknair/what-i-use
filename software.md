---
layout: default
title: Software
---

# Software List

A running list of the software I use, so setting up a new machine is just
"read this page and install everything."

## How to use this

- **Adding something**: add a row to the table below, in the right category.
- **Setting up a new machine**: go category by category, run the install command for your OS.
- **Removing something**: delete its row when you stop using it.

That's it — no scripts, no dependencies, just a markdown file.

---

Package manager assumptions: `brew` (macOS), `apt` (Linux), `winget` (Windows).
Swap for your distro/manager as needed.

## CLI Tools

| Software | Why | macOS | Linux | Windows |
|---|---|---|---|---|
| git | version control | `brew install git` | `apt install git` | `winget install Git.Git` |
| [mac-cleanup](https://github.com/mac-cleanup/mac-cleanup-py) | clean up macOS junk files | `brew install mac-cleanup` | N/A | N/A |

## Apps

| Software | Why | macOS | Linux | Windows |
|---|---|---|---|---|
| [Stirling PDF](https://github.com/Stirling-Tools/stirling-pdf) | PDF editing (self-hosted) | `docker run -p 8080:8080 frooodle/s-pdf` | `docker run -p 8080:8080 frooodle/s-pdf` | `docker run -p 8080:8080 frooodle/s-pdf` |
| [Obsidian](https://obsidian.md) | note taking | `brew install --cask obsidian` | Download from obsidian.md | `winget install Obsidian.Obsidian` |
| [Google Drive](https://drive.google.com) | cloud storage & file sync | `brew install --cask google-drive` | Download from drive.google.com | `winget install Google.GoogleDrive` |
| [Tailscale](https://tailscale.com) | VPN / Magic DNS for remote access | `brew install --cask tailscale` | `curl -fsSL https://tailscale.com/install.sh \| sh` | `winget install Tailscale.Tailscale` |
| [Nextcloud](https://nextcloud.com) | self-hosted private cloud storage (on Jetson Nano) | Access via browser at `https://dipinano.tail38e261.ts.net:8080` | `docker run -d -p 8080:80 -v /mnt/ssd/nextcloud:/var/www/html nextcloud:latest` | Access via browser at `https://dipinano.tail38e261.ts.net:8080` |

## Dev Environment

| Software | Why | macOS | Linux | Windows |
|---|---|---|---|---|
| [VS Code](https://code.visualstudio.com) | code editor | `brew install --cask visual-studio-code` | `snap install code --classic` | `winget install Microsoft.VisualStudioCode` |

## Browser Extensions / Accounts / Other

- [Raindrop.io](https://raindrop.io) — bookmarking
