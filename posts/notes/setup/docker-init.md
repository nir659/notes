---
title: Docker install scripts for Debian and Ubuntu
slug: docker-install-scripts-for-debian-and-ubuntu
type: runbook
status: draft
date: undated
updated: 2026-04-17
tags:
  - setup
  - docker
  - debian
  - ubuntu
summary: This note captures two Docker installation scripts for Debian and Ubuntu that add the official repository, install the engine stack, and add the invoking user to the docker group.
verification_status: partial
verified_on:
---

# Docker install scripts for Debian and Ubuntu

## Context

This note stores two shell scripts for installing Docker from the official repository on Debian and Ubuntu.

Both scripts follow the same shape:

- require root
- install prerequisites
- add Docker’s GPG key and APT repository
- install the engine, CLI, containerd, buildx, and compose plugin
- add the invoking non-root user to the `docker` group when possible
- verify the install with `docker run hello-world`

## Goal

- keep a reproducible Docker install path for Debian and Ubuntu
- use the official Docker repository instead of distro-default packages
- end with a quick functional verification of the install

## Environment

- Debian or Ubuntu system with `apt`
- root privileges
- outbound network access for repository bootstrap
- optional `SUDO_USER` for post-install group membership

## The Final State

If the scripts complete successfully:

- Docker packages are installed from Docker’s official repository
- `docker` commands are available
- the invoking user is added to the `docker` group when applicable
- `docker run hello-world` succeeds

## Implementation

### Debian

```bash
# debian
# /bin/bash

set -e
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'
info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}
warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}
error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
    exit 1
}

# --- Main Installation Logic ---

if [ "$(id -u)" -ne 0 ]; then
    error "This script must be run with root privileges. Please use sudo."
fi

info "Starting Docker installation process..."

info "Updating package list and installing prerequisites..."
apt-get update
apt-get install -y ca-certificates curl

info "Setting up Docker's GPG key..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

info "Adding Docker repository to APT sources..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

info "Updating package list again..."
apt-get update

info "Installing Docker Engine, CLI, and Containerd..."
 sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

if [ -n "$SUDO_USER" ]; then
    info "Adding user '$SUDO_USER' to the docker group..."
    usermod -aG docker "$SUDO_USER"
    warn "You will need to log out and log back in for the group change to take effect."
else
    warn "No non-root user detected. Skipping adding user to docker group."
fi

# --- Finalization ---

info "Docker installation is complete."
info "Verifying Docker installation..."
docker run hello-world

echo -e "\n${GREEN}Script finished successfully.${NC}"
```

### Ubuntu

```bash
# ubuntu
#!/bin/bash

set -e

# --- Color Definitions for Output ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# --- Logging Functions ---
info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}
warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}
error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
    exit 1
}

if [ "$(id -u)" -ne 0 ]; then
    error "This script must be run with root privileges. Please use sudo."
fi

info "Starting Docker installation process..."

info "Updating package list and installing prerequisites..."
apt-get update
apt-get install -y ca-certificates curl software-properties-common

info "Setting up Docker's GPG key..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

info "Adding Docker repository to APT sources..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

info "Updating package list with Docker repository..."
apt-get update

info "Installing Docker Engine, CLI, Containerd, and Docker Compose plugin..."
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

if [ -n "$SUDO_USER" ]; then
    info "Adding user '$SUDO_USER' to the docker group..."
    usermod -aG docker "$SUDO_USER"
    warn "You will need to log out and log back in for the group change to take effect."
else
    warn "No non-root user detected. Skipping adding user to docker group."
fi

info "Docker installation is complete."
info "Verifying Docker installation by running the 'hello-world' container..."
docker run hello-world

echo -e "\n${GREEN}Script finished successfully.${NC}"
```

## Verification

Expected proof after running:

- `docker --version` works
- `docker info` returns daemon information
- `docker run hello-world` succeeds
- the invoking user is present in the `docker` group after a new login session

## Failure Modes Worth Caring About

- network or repository bootstrap failure leaving half-configured APT state
- adding the repository correctly but forgetting to refresh package indexes
- assuming `SUDO_USER` exists when the script is run directly as root
- forgetting that docker group membership requires a new login session
