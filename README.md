# Stable Diffusion AUTOMATIC1111 Debian 13 Installer

Automated bash script for installing [AUTOMATIC1111's Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui) on **Debian 13 (Trixie / netinstall)** with NVIDIA GPU support.

Debian 13 ships Python 3.13 by default, which is incompatible with AUTOMATIC1111's pinned dependencies (e.g. `torch==2.1.2`, `Pillow==9.5.0`, `tokenizers`). This script installs **Python 3.11.11 side by side via [pyenv](https://github.com/pyenv/pyenv)**, without touching the system Python, and proactively sets up the virtual environment, torch, and CLIP — working around several known, reproducible installation failures along the way.

## What this script does

1. Verifies the NVIDIA driver is present (`nvidia-smi`). If not found, warns the user and waits up to 10 seconds for an explicit confirmation to continue anyway (CPU-only); aborts on timeout, no input, or a non-"y" answer
2. Asks whether pyenv/Python should be installed system-wide (`/opt/pyenv`, readable by all users) or per-user (`$HOME/.pyenv`) — relevant if the WebUI might later run under a different Linux user
3. Checks available RAM (locale-independent, with a fallback if the `available` column is missing from `free`). Only if it's below 8 GB does it ask (10s timeout, defaults to yes) whether to redirect `TMPDIR`/pip cache to `/var/tmp`, so pip/torch downloads and build artifacts don't fill up RAM on a tmpfs-backed `/tmp`. Skipped entirely on systems with 8 GB+ RAM available
4. Asks (10s timeout, defaults to yes) whether the systemd service (set up later) should be enabled for automatic start on boot
5. Installs required system packages (build tools, git, etc.)
6. Installs pyenv (if not already present), in the chosen scope
7. Builds Python 3.11.11 via pyenv, side by side with the system Python
8. Clones `AUTOMATIC1111/stable-diffusion-webui`
9. Writes `webui-user.sh` with the pyenv Python path, sane launch args (`--medvram --xformers --listen --port 7860`), and a working fork for the Stable Diffusion base repo
10. Proactively creates the venv and installs torch/torchvision and CLIP manually, before the first `webui.sh` run
11. Writes a systemd unit file (`stable-diffusion-webui.service`) so the WebUI can be managed via `systemctl` afterwards, and enables it for autostart if confirmed in step 4
12. Launches `webui.sh` once in the foreground for the initial setup and verification; use the systemd service for all later starts

## Known issues this script works around

| Issue | Cause | Fix applied by the script |
|---|---|---|
| `torch==2.1.2` has no wheel for Python 3.13 | Debian 13's default Python is too new for A1111's pins | Installs Python 3.11.11 via pyenv instead |
| `ModuleNotFoundError: No module named 'pkg_resources'` while installing CLIP | Modern `setuptools` removed `pkg_resources`; the old (2021) CLIP package still needs it during its isolated pip build | Installs `setuptools<81` + `wheel`, then installs CLIP with `--no-build-isolation` *before* `webui.sh` runs, so AUTOMATIC1111's own `is_installed("clip")` check skips its fragile install step entirely |
| `A module that was compiled using NumPy 1.x cannot be run in NumPy 2.x` | `torch==2.1.2` was compiled against NumPy 1.x | Pins `numpy<2` |
| Git clone of `Stability-AI/stablediffusion` asks for a GitHub login / fails with 401 | The original repository was removed/privatized from public GitHub (a confirmed, global issue affecting all fresh AUTOMATIC1111 installs since ~December 2025) | Sets `STABLE_DIFFUSION_REPO` to `w-e-w/stablediffusion.git`, the fork recommended by the AUTOMATIC1111 project itself (used on its `dev` branch) |
| RAM check silently reports empty/wrong values on non-English locales | `free`'s row label is translated (e.g. German locales show `Speicher:` instead of `Mem:`), so pattern matching on `Mem:` fails | Runs `free -m` with `LC_ALL=C` to force the English row label regardless of system locale, with a fallback to the `free` column if the `available` column is missing (older `procps` versions) |

## Prerequisites

- Debian 13 (Trixie), netinstall or full install
- NVIDIA driver already installed (`nvidia-smi` must return GPU info)
- `sudo` privileges (required for `apt install` steps)
- Internet access to GitHub, PyPI, and the PyTorch package index

## Usage

```bash
git clone https://github.com/AIScriptHub/stable-diffusion_AUTOMATIC1111_debian-13_install.git
cd stable-diffusion_AUTOMATIC1111_debian-13_install
bash stable-diffusion_AUTOMATIC1111_debian-13_install_v0.4.sh
```

The script is idempotent: steps that were already completed in a previous run (pyenv, Python 3.11.11, the repository clone, the venv) are automatically skipped on subsequent runs.

## Interactive prompts

The script asks a few yes/no questions during the run. Each has a 10-second timeout:

| Prompt | Default on timeout/no input | Purpose |
|---|---|---|
| Continue without an NVIDIA driver? | **No** (aborts) | Safety default — CPU-only is not what this script's CUDA setup targets |
| System-wide or per-user pyenv install? | *(no timeout — waits for `1` or `2`)* | See below |
| Redirect TMPDIR/pip cache to `/var/tmp`? | **Yes** (only asked if available RAM < 8 GB) | Prevents pip/torch build files from filling up RAM-backed `/tmp` |
| Enable systemd autostart on boot? | **Yes** | Whether the WebUI should start automatically after a reboot |

### pyenv installation scope

- **System-wide** (`/opt/pyenv`, requires sudo): recommended if the WebUI might later run under a different Linux user. Environment is exposed via `/etc/profile.d/pyenv.sh`, readable by all users. Per-user `venv`s are still required for each user — only the Python interpreter itself is shared.
- **User-space** (`$HOME/.pyenv`): simpler, no system-wide changes, but the `python_cmd` path in `webui-user.sh` only resolves for the user who ran the script.

## systemd service

The script writes `/etc/systemd/system/stable-diffusion-webui.service`, so the WebUI can be managed like any other system service:

```bash
sudo systemctl start   stable-diffusion-webui
sudo systemctl stop    stable-diffusion-webui
sudo systemctl status  stable-diffusion-webui
sudo systemctl enable  stable-diffusion-webui   # start automatically on boot
journalctl -u stable-diffusion-webui -f          # follow logs
```

`webui-user.sh` still controls `python_cmd`, `COMMANDLINE_ARGS`, and `STABLE_DIFFUSION_REPO` — the service just calls `webui.sh`, the same as running it manually would. If a TMPDIR redirect was applied (see above), the service's `Environment=` lines carry it over so it also applies to later model downloads and image generation, not just the initial install.

## LAN access

By default the web UI only listens on `127.0.0.1`. The script's generated `webui-user.sh` already includes `--listen --port 7860`, which makes the UI reachable from other devices on the LAN at `http://<lan-ip>:7860`.

> **Security note:** `--listen` exposes the web UI to the entire LAN without authentication. If the network isn't fully trusted, add `--gradio-auth username:password` to `COMMANDLINE_ARGS` in `webui-user.sh`.

## Tested on

- Debian 13 netinstall, ~6 GB RAM, NVIDIA GPU (driver already installed) — with the TMPDIR redirect prompt
- Debian 13, ~8 GB RAM, NVIDIA GPU — with the TMPDIR redirect skipped automatically

## Re-running / troubleshooting

If a run fails partway through, simply re-run the script — completed steps are skipped. If the `venv` directory ends up in a broken state (e.g. torch installed but CLIP missing), remove it and re-run:

```bash
rm -rf ~/stable-diffusion-webui/venv
bash Installer_-_Debian-13_Stable-Diffusion_AUTOMATIC1111_v0.4.sh
```
## Disclaimer

This script is provided as-is, without warranty. It automates commands based on a working, real-world installation on Debian 13; results may vary depending on hardware and system configuration.

---

## Support this project

If this script saved you time, consider supporting further development:

| Currency | Address |
|---|---|
| BTC | `your-btc-address-here` |
| ETH | `your-eth-address-here` |

Thank you for your support! ⭐

---

## ❤️ Support the Project

This script was created and published free of charge for the open source community. If you find it useful and would like to support future development, consider making a small donation:

| Cryptocurrency            | Address                                                                                                   |
| ------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Bitcoin (BTC)**         | `33AXe8Z8XBuGKx9eHHmGnvbawrNYjSgDcM`                                                                      |
| **Ethereum (ETH)**        | `0xa61d178EA84C2200A8617b51B4bCf98F87ff59Ff`                                                              |
| **Solana (SOL)**          | `BDf5EgsN8fRUicYzeM8cuaNhL7zdty2qsEj2mC2jA4Fm`                                                            |
| **Ripple (XRP)**          | `rLHzPsX6oXkzU2qL12kHCH8G8cnZv1rBJh`                                                                      |
| **Cardano (ADA)**         | `addr1q8anur2wvvc6pv3cpp30vv05makyra8huh0lk0yhdk6hcnlrzr27g03klu862usxqsru794d03gzkk8n86ta34n85z0svn5ams` |
| **Tether (USDT - ERC20)** | `0xa61d178EA84C2200A8617b51B4bCf98F87ff59Ff`                                                              |

Thank you for your support! 🙏

---

<p align="center">
  Created and maintained by <b>Speefak</b>
</p>

This script is provided as-is, without warranty. It automates commands based on a working, real-world installation on Debian 13; results may vary depending on hardware and system configuration.
