# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **dockur/windows**: a Docker container that runs Windows using QEMU/KVM virtualization with automatic ISO downloading, installation, and web-based viewer access.

The container provides a complete Windows virtualization environment with:
- Automatic Windows ISO download via Mido scripts
- Unattended installation orchestration
- KVM acceleration for performance
- Web viewer on port 8006 (noVNC)
- RDP access on port 3389
- Samba file sharing via `/shared` volume
- Support for Windows versions from 2000 to Server 2025

## Core Architecture

### Boot Sequence (src/entry.sh)

The container follows a strict initialization sequence:
1. `start.sh` - Startup hook
2. `utils.sh` - Core functions library
3. `reset.sh` - System initialization
4. `server.sh` - Web server startup
5. `define.sh` - Version parsing and validation (~1800 lines)
6. `mido.sh` - ISO download orchestration
7. `install.sh` - Automated Windows installation
8. `disk.sh` - Disk initialization
9. `display.sh` - Graphics setup
10. `network.sh` - Network configuration
11. `samba.sh` - Shared folder setup
12. `boot.sh` - Boot configuration
13. `proc.sh` - CPU configuration
14. `power.sh` - Shutdown handling
15. `memory.sh` - RAM validation
16. `config.sh` - QEMU argument assembly
17. `finish.sh` - Finalization

Each script is sourced sequentially and handles a specific initialization domain. Changes to boot logic should maintain this ordering.

### Version Management (src/define.sh)

The `parseVersion()` function maps user-friendly version strings (e.g., `"11"`, `"win10"`, `"2022"`) to internal version identifiers (e.g., `"win11x64"`, `"win2022-eval"`).

Key patterns:
- Desktop: `win11x64`, `win10x64`, `win81x64`, `win7x64`, etc.
- Server: `win2025-eval`, `win2022-eval`, `win2019-eval`, etc.
- Enterprise/LTSC editions have `-enterprise-eval` or `-ltsc-eval` suffixes
- Legacy systems: `winvistax64`, `winxpx86`, `win2kx86`

When adding new Windows versions, update the case statement in `parseVersion()` following the established naming convention.

### ISO Download (src/mido.sh)

Downloads are handled by curl with extensive error handling via `handle_curl_error()`. The script:
- Supports multiple mirror sources (MIRRORS=4)
- Implements retry logic for network failures
- Validates downloaded ISO integrity
- Falls back to manual installation if download fails
- Uses user-agent rotation to avoid rate limiting

### Installation System (src/install.sh)

Orchestrates automated Windows installation:
- **Backup**: Previous installations are backed up to `/storage/backups` when VERSION changes
- **Skip Logic**: `skipInstall()` detects existing installations and avoids reinstallation
- **Unattended Install**: Injects autounattend.xml for zero-touch deployment
- **Driver Injection**: Virtio drivers from `/var/drivers.txz` are integrated during install

The installation process modifies ISOs in-place using `wimtools` and `genisoimage`.

## Development Commands

### Building

```bash
docker build -t dockurr/windows .
```

Build uses multi-stage Dockerfile with architecture-specific stages:
- `build-amd64`: x86_64 images (base: qemux/qemu:7.27)
- `build-arm64`: ARM64 images (base: dockurr/windows-arm)

### Testing

Shellcheck validation runs in CI:
```bash
# Local shellcheck (if needed)
shellcheck src/*.sh
```

The `.github/workflows/check.yml` workflow runs shellcheck on all shell scripts before builds.

### Running Locally

```bash
docker compose up
```

This uses `compose.yml` with default settings:
- VERSION: "11" (Windows 11 Pro)
- Web viewer: http://localhost:8006
- RDP: localhost:3389
- Storage: `./windows` directory

### Testing Version Changes

To test different Windows versions:
```bash
VERSION="10" docker compose up
VERSION="2022" docker compose up
VERSION="win7x64-ultimate" docker compose up
```

## Key Files and Paths

### Container Paths
- `/run/` - All source scripts are copied here at runtime
- `/storage/` - Persistent volume for VM disk and state
- `/storage/windows.qcow2` - Main VM disk image
- `/storage/windows.iso` - Downloaded Windows ISO (if custom)
- `/storage/windows.base` - Tracks which ISO version is installed
- `/storage/backups/` - Previous installation backups
- `/shared/` - Host-to-guest file sharing directory
- `/var/drivers.txz` - Virtio driver archive

### Source Structure
- `src/` - All shell scripts (4634 total lines)
- `assets/` - Binary assets and tools
- `Dockerfile` - Multi-arch container definition
- `compose.yml` - Default Docker Compose configuration

## Environment Variables

Critical variables (defaults in Dockerfile):
- `VERSION="11"` - Windows version identifier
- `RAM_SIZE="4G"` - Guest RAM allocation
- `CPU_CORES="2"` - Guest CPU count
- `DISK_SIZE="64G"` - Virtual disk size

Additional variables (runtime):
- `MANUAL="Y"` - Skip automatic installation
- `USERNAME="Docker"` - Windows username (default)
- `PASSWORD="admin"` - Windows password (default)
- `LANGUAGE="English"` - Installation language
- `KEYBOARD="en-US"` - Keyboard layout
- `DHCP="Y"` - Enable DHCP networking for guest

## QEMU/KVM Integration

The container requires:
- `/dev/kvm` device access for hardware acceleration
- `/dev/net/tun` for networking
- `NET_ADMIN` capability for network bridge setup
- `/dev/vhost-net` (optional, for DHCP mode)

QEMU arguments are assembled in `config.sh` and executed in `entry.sh`. The VM output is logged to `$QEMU_LOG` and terminal output to `$QEMU_TERM`.

## Architecture Patterns

### Error Handling
- All scripts use `set -Eeuo pipefail` for strict error propagation
- Functions return non-zero on failure
- `error()` utility function provides consistent logging
- Critical failures exit with specific codes (e.g., exit 15 for QEMU failures)

### State Management
- Installation state is tracked via marker files in `/storage/`
- `windows.base` contains the installed ISO identifier
- `windows.boot` indicates boot mode
- State changes trigger backup workflows

### Modular Script Loading
- Each domain script is sourced in sequence
- Scripts communicate via environment variables
- No direct script-to-script sourcing (except utils.sh)
- Central orchestration in entry.sh

## Common Modifications

### Adding a New Windows Version
1. Update `parseVersion()` in `src/define.sh` with new case statement
2. Add download URL mapping if using Mido
3. Test with `VERSION="new-version"` environment variable
4. Update README.md version table

### Modifying Installation Behavior
- Edit `src/install.sh` for installation logic changes
- Modify autounattend.xml template if present in assets/
- Adjust driver injection in the install script

### Changing QEMU Configuration
- Edit `src/config.sh` for QEMU argument assembly
- Modify `src/proc.sh` for CPU configuration
- Edit `src/memory.sh` for RAM validation logic
- Update `src/display.sh` for graphics changes

## CI/CD

GitHub Actions workflows:
- `.github/workflows/build.yml` - Main build and publish
- `.github/workflows/check.yml` - Shellcheck validation
- `.github/workflows/test.yml` - Integration testing
- `.github/workflows/hub.yml` - Docker Hub sync
- `.github/workflows/review.yml` - PR validation

Builds publish to:
- Docker Hub: `dockurr/windows`
- GitHub Container Registry: `ghcr.io/dockur/windows`

## Debugging

### View QEMU Logs
```bash
docker exec windows cat /var/log/qemu.log
```

### Check Installation Progress
Web viewer shows real-time installation: http://localhost:8006

### Access Container Shell
```bash
docker exec -it windows bash
```

### Verify KVM
```bash
docker exec windows ls -l /dev/kvm
```

## Notes

- The project uses QEMU 7.27 from the `qemux/qemu` base image
- Virtio drivers version 1.9.48 are automatically injected
- ARM64 support is provided via `dockurr/windows-arm` base
- The container must run with `stop_grace_period: 2m` to allow clean Windows shutdown
- Samba (wsdd-native) enables Windows network discovery of the shared folder
