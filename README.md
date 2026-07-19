# Stable Diffusion AUTOMATIC1111 Debian 13 Installer

Automated bash script for installing [AUTOMATIC1111's Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui) on **Debian 13 (Trixie / netinstall)** with NVIDIA GPU support.

Debian 13 ships Python 3.13 by default, which is incompatible with AUTOMATIC1111's pinned dependencies (e.g. `torch==2.1.2`, `Pillow==9.5.0`, `tokenizers`). This script installs **Python 3.11.11 side by side via [pyenv](https://github.com/pyenv/pyenv)**, without touching the system Python, and proactively sets up the virtual environment, torch, and CLIP — working around several known, reproducible installation failures along the way.

## What this script does

1. Verifies the NVIDIA driver is present (`nvidia-smi`)
2. Redirects `TMPDIR` / pip cache to disk (important on low-RAM systems, so downloads and build artifacts don't end up on a RAM-backed `/tmp`)
3. Installs required system packages (build tools, git, etc.)
4. Installs pyenv (if not already present)
5. Builds Python 3.11.11 via pyenv, side by side with the system Python
6. Clones `AUTOMATIC1111/stable-diffusion-webui`
7. Writes `webui-user.sh` with the pyenv Python path, sane launch args, and a working fork for the Stable Diffusion base repo
8. Proactively creates the venv and installs torch/torchvision and CLIP manually, before the first `webui.sh` run
9. Launches `webui.sh`, which installs only the remaining, unproblematic requirements and starts the web UI on `http://127.0.0.1:7860` (also reachable on the LAN, see below)

## Known issues this script works around

| Issue | Cause | Fix applied by the script |
|---|---|---|
| `torch==2.1.2` has no wheel for Python 3.13 | Debian 13's default Python is too new for A1111's pins | Installs Python 3.11.11 via pyenv instead |
| `ModuleNotFoundError: No module named 'pkg_resources'` while installing CLIP | Modern `setuptools` removed `pkg_resources`; the old (2021) CLIP package still needs it during its isolated pip build | Installs `setuptools<81` + `wheel`, then installs CLIP with `--no-build-isolation` *before* `webui.sh` runs, so AUTOMATIC1111's own `is_installed("clip")` check skips its fragile install step entirely |
| `A module that was compiled using NumPy 1.x cannot be run in NumPy 2.x` | `torch==2.1.2` was compiled against NumPy 1.x | Pins `numpy<2` |
| Git clone of `Stability-AI/stablediffusion` asks for a GitHub login / fails with 401 | The original repository was removed/privatized from public GitHub (a confirmed, global issue affecting all fresh AUTOMATIC1111 installs since ~December 2025) | Sets `STABLE_DIFFUSION_REPO` to `w-e-w/stablediffusion.git`, the fork recommended by the AUTOMATIC1111 project itself (used on its `dev` branch) |

## Prerequisites

- Debian 13 (Trixie), netinstall or full install
- NVIDIA driver already installed (`nvidia-smi` must return GPU info)
- `sudo` privileges (required for `apt install` steps)
- Internet access to GitHub, PyPI, and the PyTorch package index

## Usage

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
bash SD-Automatic1111_install_D13.sh
```

The script is idempotent: steps that were already completed in a previous run (pyenv, Python 3.11.11, the repository clone, the venv) are automatically skipped on subsequent runs.

## LAN access

By default the web UI only listens on `127.0.0.1`. The script's generated `webui-user.sh` already includes `--listen --port 7860`, which makes the UI reachable from other devices on the LAN at `http://<lan-ip>:7860`.

> **Security note:** `--listen` exposes the web UI to the entire LAN without authentication. If the network isn't fully trusted, add `--gradio-auth username:password` to `COMMANDLINE_ARGS` in `webui-user.sh`.

## Tested on

- Debian 13 netinstall, 6 GB RAM, NVIDIA GPU (driver already installed)

## Re-running / troubleshooting

If a run fails partway through, simply re-run the script — completed steps are skipped. If the `venv` directory ends up in a broken state (e.g. torch installed but CLIP missing), remove it and re-run:

```bash
rm -rf ~/stable-diffusion-webui/venv
bash SD-Automatic1111_install_D13.sh
```

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
