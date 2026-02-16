---
author: Darryl Buswell
pubDatetime: 2026-02-12T12:00:00Z
modDatetime: 2026-02-12T12:00:00Z
title: "Automating TrueNAS VM Deployments"
slug: automating-truenas-vm-deployments
featured: false
draft: false
tags:
  - truenas
  - automation
  - shell
  - devops
  - weekendproject
description: "A deep dive into using shell scripts, midclt, and cloud-init to automate VM deployments on TrueNAS."
---

I decided to rebuild a couple of VMs on my TrueNAS machine last weekend. But rather than doing it the manual way, I figured it was time to get automating. The goal was simple: create a fully automated, repeatable process for building and provisioning a Debian VM on TrueNAS.

## The Magic Behind the Curtain: `midclt`

The secret sauce that makes this all possible is `midclt`, the command-line interface for the TrueNAS middleware daemon. While the UI is great for occasional use, `midclt` is the key to automation on TrueNAS. Nearly everything you can do in the UI, you can do with a `midclt call`.

## The Three Scripts

The work I did here is broken down into three main scripts:

1.  `push-os-image.sh`: Downloads a Debian cloud image and gets it onto the TrueNAS server.
2.  `push-seed-iso.sh`: Creates a `cloud-init` seed ISO to pre-configure the VM on first boot.
3.  `deploy-vm.sh`: The main script which uses `midclt` to create and configure the VM from all the prepared parts.

---

### Part 1: Pushing the OS Image

First things first, you need the base OS image on your TrueNAS server. This script handles downloading a Debian cloud image and copying it to the right dataset on the TrueNAS host.

`push-os-image.sh`
```bash
#!/bin/bash

# Configuration
IMG_VERSION="20251117-2299"
IMG_NAME="debian-13-generic-amd64-$IMG_VERSION.qcow2"
IMG_URL="https://cloud.debian.org/images/cloud/trixie/$IMG_VERSION/$IMG_NAME"
REMOTE_HOST="your-truenas-ip"
REMOTE_USER="root"
REMOTE_DIR="/mnt/your-pool/vm/image/"

BASE_DIR="$(cd "$(dirname "$0")" && pwd)"
IMAGES_DIR="$BASE_DIR/../_images"
IMAGE_PATH="$IMAGES_DIR/$IMG_NAME"

# 1. Setup local images directory
echo "Ensuring local images directory exists..."
mkdir -p "$IMAGES_DIR"

# 2. Download Debian cloud image (if missing)
if [ ! -f "$IMAGE_PATH" ]; then
    echo "Downloading $IMG_NAME..."
    wget -O "$IMAGE_PATH" "$IMG_URL"
else
    echo "Image $IMG_NAME already exists locally."
fi

# 3. Create remote directory
echo "Ensuring remote directory exists on $REMOTE_HOST..."
ssh "$REMOTE_USER@$REMOTE_HOST" "mkdir -p $REMOTE_DIR"

# 4. Upload image
echo "Uploading $IMG_NAME to $REMOTE_HOST..."
scp "$IMAGE_PATH" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR"

echo "Done."
```

### Part 2: Seeding the Configuration with `cloud-init`

`cloud-init` is the standard for early-stage virtual machine configuration. By creating a special seed ISO, we can tell the VM how to configure itself on its very first boot. This includes setting the hostname, creating user accounts, adding SSH keys, configuring network interfaces, and more.

Our `push-seed-iso.sh` script handles the generation of this ISO and uploads it to TrueNAS.

`push-seed-iso.sh`
```bash
#!/bin/bash

# Configuration
REMOTE_HOST="your-truenas-ip"
REMOTE_USER="root"
REMOTE_ISO_DIR="/mnt/your-pool/vm/iso/"
SEED_ISO_NAME="your-vm-seed.iso"

BASE_DIR="$(cd "$(dirname "$0")" && pwd)"
CLOUD_INIT_DIR="$BASE_DIR/configs/cloud-init"

# 0. Check for local dependencies
if ! command -v cloud-localds &> /dev/null; then
  echo "Installing missing dependency: cloud-image-utils..."
  sudo apt-get update && sudo apt-get install -y cloud-image-utils
fi

# 1. Generate cloud-init seed ISO
echo "Generating seed ISO..."
cd "$CLOUD_INIT_DIR" || exit
if [ -f "generate-seed.sh" ]; then
    bash generate-seed.sh
else
    echo "Error: generate-seed.sh not found in $CLOUD_INIT_DIR"
    exit 1
fi

# 2. Create remote directory
echo "Ensuring remote directory exists on $REMOTE_HOST..."
ssh "$REMOTE_USER@$REMOTE_HOST" "mkdir -p $REMOTE_ISO_DIR"

# 3. Upload Seed ISO
echo "Uploading seed.iso as $SEED_ISO_NAME to $REMOTE_HOST..."
scp "seed.iso" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_ISO_DIR$SEED_ISO_NAME"

echo "Done."
```

### Part 3: The Grand Finale: Deploying the VM

This is the script that does the heavy lifting. It connects to the TrueNAS server via SSH and orchestrates a series of `midclt` calls to build our VM.

Here's a breakdown of what it does:
1.  **Checks for an existing VM** with the same name.
2.  **Creates a ZFS volume (Zvol)** to act as the VM's main disk.
3.  **Converts the `.qcow2` image** we uploaded into a raw format directly onto the Zvol.
4.  **Creates a basic VM** with our desired CPU, memory, and bootloader settings.
5.  **Attaches all the devices**: the Zvol as a VirtIO disk, a SPICE display for VNC access, two network interfaces, and finally, our `cloud-init` seed ISO as a CD-ROM.
6.  **Starts the VM**.

`deploy-vm.sh`
```bash
#!/bin/bash

# Configuration
VM_NAME="my-new-vm"
TRUENAS_HOST="your-truenas-ip"
TRUENAS_USER="root" # Note: Using root not a good idea for production.
ZVOL_PATH="your-pool/vm/zvol/$VM_NAME"
IMG_VERSION="20251117-2299"
IMG_PATH="/mnt/your-pool/vm/image/debian-13-generic-amd64-$IMG_VERSION.qcow2"
SEED_ISO_PATH="/mnt/your-pool/vm/iso/your-vm-seed.iso"

echo "========================================"
echo "Deploying VM: $VM_NAME on $TRUENAS_HOST"
echo "========================================"

# 1. Check if VM already exists
echo ">>> Step 1: Checking for existing VM..."
EXISTING_VM_ID=$(ssh "$TRUENAS_USER@$TRUENAS_HOST" "midclt call vm.query '[[\"name\", \"=\", \"$VM_NAME\"]]' | sed -E 's/.*\"id\": *([0-9]+).*/\1/' | head -n1")

if [[ "$EXISTING_VM_ID" =~ ^[0-9]+$ ]]; then
  echo "VM already exists (ID: $EXISTING_VM_ID). Skipping deployment."
  exit 0
fi

# 2. Ensure Seed ISO exists
echo ">>> Step 2: Ensuring Seed ISO exists..."
ssh "$TRUENAS_USER@$TRUENAS_HOST" "ls '$SEED_ISO_PATH' &>/dev/null" || { echo "Error: Seed ISO not found at $SEED_ISO_PATH"; exit 1; }

# 3. Ensure Zvol exists
echo ">>> Step 3: Ensuring Zvol exists..."
ssh "$TRUENAS_USER@$TRUENAS_HOST" "if ! ls /dev/zvol/$ZVOL_PATH &>/dev/null; then zfs create -V 20G $(dirname $ZVOL_PATH)/$VM_NAME; fi"

# 4. Convert image to Zvol
echo ">>> Step 4: Converting image to Zvol..."
ssh "$TRUENAS_USER@$TRUENAS_HOST" "qemu-img convert -p -f qcow2 -O raw '$IMG_PATH' '/dev/zvol/$ZVOL_PATH'"

# 5. Create VM shell
echo ">>> Step 5: Creating VM shell..."
RESULT=$(ssh "$TRUENAS_USER@$TRUENAS_HOST" "midclt call vm.create '{
  \"name\": \"$VM_NAME\",
  \"description\": \"My New VM\",
  \"vcpus\": 2,
  \"memory\": 15259,
  \"bootloader\": \"UEFI\",
  \"cpu_mode\": \"HOST-PASSTHROUGH\",
  \"autostart\": true,
  \"time\": \"LOCAL\"
}'")

VM_ID=$(echo "$RESULT" | sed -E 's/.*\"id\": *([0-9]+).*/\1/' | head -n1)

if [[ ! "$VM_ID" =~ ^[0-9]+$ ]]; then
  echo "Error: Failed to obtain a valid numeric VM ID. Result: $RESULT"
  exit 1
fi
echo "Created VM ID: $VM_ID"

# 6. Configure VM and devices
echo ">>> Step 6: Configuring VM devices..."
ssh "$TRUENAS_USER@$TRUENAS_HOST" "bash -s" <<EOF
  # DISK
  midclt call vm.device.create '{
    "vm": $VM_ID, 
    "order": 1001, 
    "attributes": {
      "dtype": "DISK",
      "path": "/dev/zvol/$ZVOL_PATH",
      "type": "VIRTIO"
    }
  }'

  # DISPLAY
  midclt call vm.device.create '{
    "vm": $VM_ID, 
    "order": 1002, 
    "attributes": {
      "dtype": "DISPLAY",
      "web": true, 
      "type": "SPICE", 
      "bind": "0.0.0.0", 
      "wait": false,
      "password": "your-vnc-password"
    }
  }'

  # NIC 1
  midclt call vm.device.create '{
    "vm": $VM_ID, 
    "order": 1003, 
    "attributes": {
      "dtype": "NIC",
      "type": "VIRTIO", 
      "nic_attach": "br0", 
      "mac": "00:a0:98:00:00:01"
    }
  }'

  # CDROM (Seed ISO)
  midclt call vm.device.create '{
    "vm": $VM_ID, 
    "order": 1004, 
    "attributes": {
      "dtype": "CDROM",
      "path": "$SEED_ISO_PATH"
    }
  }'

  # Start VM
  echo ">>> Starting VM..."
  midclt call vm.start $VM_ID
EOF

echo "Deployment complete."
```

---

## Gotchas & Things to Watch Out For

Building this wasn't without its quirks. Here are a few things to keep in mind if you adapt this for your own use:

*   **User Permissions**: The scripts use the `root` user for simplicity. For any real-world use, you should create a dedicated user with specific permissions or, even better, use a TrueNAS API Key and interact with the API directly.
*   **JSON in Shell Scripts**: Passing multi-line JSON to `midclt` from a shell script can be tricky. The syntax has to be perfect. I recommend testing your `midclt` commands directly on the TrueNAS server before putting them into a script.
*   **ZFS Paths**: Double-check your ZFS pool names and paths (`your-pool/vm/zvol`, etc.). These must match your TrueNAS configuration exactly.
*   **Network Interfaces**: The script attaches the VM to `br0`. This bridge must already exist on your server.

## Conclusion

I hope this quick walkthrough has been helpful and inspires the TrueNAS users out there to start automating with `midclt`. Happy scripting!
