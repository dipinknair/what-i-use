---
layout: default
title: Home
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

## Apps

| Software | Why | macOS | Linux | Windows |
|---|---|---|---|---|
| [Stirling PDF](https://github.com/Stirling-Tools/stirling-pdf) | PDF editing (self-hosted) | `docker run -p 8080:8080 frooodle/s-pdf` | `docker run -p 8080:8080 frooodle/s-pdf` | `docker run -p 8080:8080 frooodle/s-pdf` |
| [Obsidian](https://obsidian.md) | note taking | `brew install --cask obsidian` | Download from obsidian.md | `winget install Obsidian.Obsidian` |
| [Google Drive](https://drive.google.com) | cloud storage & file sync | `brew install --cask google-drive` | Download from drive.google.com | `winget install Google.GoogleDrive` |

## Dev Environment

| Software | Why | macOS | Linux | Windows |
|---|---|---|---|---|
|  |  |  |  |  |

## Browser Extensions / Accounts / Other

- [Raindrop.io](https://raindrop.io) — bookmarking
