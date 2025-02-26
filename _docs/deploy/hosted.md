---
title: Hosted Mode
category: Deploy
order: 5
---
# Hosted mode
- TOC
{:toc}

GyroidOS containers can also be run natively on your host, using the hosted mode.
This section describes how to run the hosted mode.
These instructions have been tested and work on Debian 12 testing. 

## Requirements
Have a prebuilt container image ready or build one yourself as described in [Build](/build/build).

### Required packages
Install all required packages with the following command: 
```
sudo apt update && sudo apt-get install -y git build-essential unzip re2c pkg-config \
    check lxcfs libprotobuf-c-dev automake libtool libselinux1-dev libcap-dev \
    protobuf-c-compiler libssl-dev udhcpd udhcpd libsystemd-dev debootstrap \
    python3-protobuf
```
### Protobuf-c-text
To install protobuf-c-text, run the following commands:
```
git clone https://github.com/gyroidos/external_protobuf-c-text.git
cd external_protobuf-c-text/
./autogen.sh
./configure --enable-static=yes
make 
make install
ldconfig
```

## Installation

### CML
Clone, compile and install the neccesary components with
```
git clone https://github.com/gyroidos/cml
cd cml/
SYSTEMD=y make -f Makefile_lsb 
sudo SYSTEMD=y make -f Makefile_lsb install
```

### Additional tools
Clone and install the necessary components with
```
git clone https://github.com/gyroidos/gyroidos_build
cd gyroidos_build/cml-tools
sudo make install
```

## Setup

1. Run `cml-scd` once to create the certificates in `/var/lib/cml/tokens`
2. Create `cml-control` group and add the current user to it
3. Create test certificates with `cml_gen_dev_certs /path/to/dir`. Note that the `/path/to/dir` directory should not exist.
5. Copy the `/path/to/dir/ssig_rootca.cert` to `/var/lib/cml/tokens/`
6. Start the `cmld.service`

<details markdown="0">
<summary style="display: list-item">Setup script</summary>

<pre>
#!/bin/bash

set -euo pipefail

echo "Initalizing tokens"
sudo cml-scd

# Check if cml-control group already exists
GROUP_NAME="cml-control"
if $(groups | grep -q "$GROUP_NAME"); then
    echo "Group '$GROUP_NAME' already exists"
else
    echo "Creating cml-control group"
    sudo addgroup cml-control
fi

echo "Adding current user to group"
sudo usermod -aG cml-control $(whoami)
echo "Reloading groups"
newgrp "$GROUP_NAME"

# Calling `cml_gen_dev_certs` on an existing directory does not create any certificates
if [ -d ~/test-certs ]; then
    echo "Remove the directory at '~/test-certs' and re-run the script"
    exit 1
fi

echo
echo "Creating root certificates"
cml_gen_dev_certs ~/test-certs

echo
echo "Installing root certificates"
sudo cp ~/test-certs/ssig_rootca.cert /var/lib/cml/tokens/

echo
echo "Starting cmld.service"
sudo systemctl start cmld.service

echo
echo "Use 'systemctl status cmld.service' to verify that the service is active."
</pre>
</details>

## Add a Guest OS
1. Create and enter a new folder, e.g. `~/cmld_guestos`
2. Initalize a guest os called `guest-bookworm` with `cml_build_guestos init guest-bookworm --pki ~/deps/gyroidos_build/test-certs/`
3. Create a new folder `rootfs-builder`
4. Create a new rootfs for Debian 12 Bookworm `sudo debootstrap --arch=amd64 bookworm rootfs-builder http://deb.debian.org/debian/`
5. Create an uncompressed tarball with `sudo tar -cf guest-bookworm.tar -C ./rootfs-builder .`
6. Move the tarball into `cmld_guestos/rootfs` with `mv guest-bookworm.tar  rootfs/guest-bookworm`
7. Build the guest os with `sudo cml_build_guestos build guest-bookworm`
8. I don't know `sudo cp -r out/trustx-guests/<guestos-name>os* /var/lib/cml/operatingsystems`
9. Restart `cmld` service
10. (Optional) Remove with `rootfs-builder`directory with `sudo rm -r rootfs-builder`

3. You can either push your own certificate by using cml-control push_ca or copy the ssig_rootca.cert from your build to /var/lib/cml/tokens
4. Run scd again, this time it will start a loop and keep running
5. Run cmld and keep it running
2. Install your guestos as described in [GuestOS configuration](/operate/guestos_config)
6. You can now use control as described in [Basic Operation](/operate/control)

