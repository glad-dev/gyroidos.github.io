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
These instructions have been tested and work on Debian 13 Trixi.

## Requirements
> cml requires OpenSSL version >= 3.2

Have a prebuilt container image ready or build one yourself as described in [Build](/build/build).

### Required packages
Install all required packages with the following command:
```
sudo apt update && sudo apt-get install -y git build-essential unzip re2c pkg-config \
    check lxcfs libprotobuf-c-dev automake libtool libselinux1-dev libcap-dev \
    protobuf-c-compiler libssl-dev udhcpd udhcpd libsystemd-dev debootstrap \
    squashfs-tools python3-protobuf protobuf-compiler \
    cryptsetup-bin iptables
```

### Protobuf-c-text
To install protobuf-c-text, run the following commands:
```
git clone https://github.com/gyroidos/external_protobuf-c-text.git
cd external_protobuf-c-text/
./autogen.sh
./configure --enable-static=yes
make
sudo make install
sudo ldconfig
```

## Installation

### CML
Clone, compile and install the neccesary components with
```
git clone https://github.com/gyroidos/cml
cd cml/
sed -i 's/IDMAPPED ?= y/IDMAPPED ?= n/' daemon/Makefile
SYSTEMD=y make -f Makefile_lsb
sudo SYSTEMD=y make -f Makefile_lsb install
```

### Additional tools
Clone and install the necessary components with
```
git clone https://github.com/gyroidos/gyroidos_build
cd gyroidos_build/cml_tools
sudo make install
```

## Setup

### Automatic

Download and run the [setup script](/assets/hosted-setup.sh) to automatically perform the steps outlined below.

### Manually

1. Create the log directory `/var/log/cml/`
2. Run `sudo cml-scd` once to create certificates
3. Create the `cml-control` group and add the current user to it
4. Create test certificates with `cml_gen_dev_certs /path/to/dir`. Note that the `/path/to/dir` directory should not exist.
5. Copy the `/path/to/dir/ssig_rootca.cert` to `/var/lib/cml/tokens/`
6. Start the `cmld` systemd service


## Add a Guest OS

### Manually

1. Create and enter a new folder, e.g. `~/cmld_guestos`
2. Initalize a guest os called `guest-bookworm` with `cml_build_guestos init guest-bookworm --pki ~/test-certs/`
3. Create a new folder `rootfs-builder`
4. Create a new rootfs for Debian 12 Bookworm `sudo debootstrap bookworm rootfs-builder http://deb.debian.org/debian/`
5. Create an uncompressed tarball with `sudo tar -cf guest-bookworm.tar -C ./rootfs-builder .`
6. Move the tarball into `cmld_guestos/rootfs` with `mv guest-bookworm.tar  rootfs/guest-bookwormos.tar`
7. Build the guest os with `sudo cml_build_guestos build guest-bookworm`
8. I don't know `sudo cp -r out/gyroidos-guests/* /var/lib/cml/operatingsystems`
9. Restart `cmld` service
10. Verify that `cml-control list_guestos` contains `guest-bookworm`
11. Create container with `cml-control create conf/guest-bookwormcontainer.conf`
12. Add `signed_configs: false` to `/etc/cml/device.conf`
13. Change container password `cml-control change_pin "guest-bookwormcontainer"`. Default password is "trustme"
14. Restart `cmld` service
15. Start the container with `cml-control start "guest-bookwormcontainer"`. If command returns `CONTAINER_START_EINTERNAL`, run it again.
16. Verify that it is running with `    `
17. In `ps auxf`, look for `/sbin/init` with weird owner
18. Connect to it using `sudo nsenter -at $PID`

<details markdown="0">
<summary style="display: list-item">GuestOS script</summary>

<pre>
#!/bin/bash

set -euo pipefail

if [ ! -d ~/test-certs/ ]; then
    echo "Certificate directory '~/test-certs/' is missing"
    exit 1
fi

echo "Creating GuestOS directory"
mkdir ~/cmld_guestos
cd ~/cmld_guestos

echo
echo "Initalizing guest os"
cml_build_guestos init guest-bookworm --pki ~/test-certs/

echo
echo "Creating debian 12 rootfs"
mkdir rootfs-builder
sudo debootstrap bookworm rootfs-builder http://deb.debian.org/debian/

echo
echo "Creating tar archive of rootfs"
sudo tar -cf guest-bookworm.tar -C ./rootfs-builder .
mv guest-bookworm.tar  rootfs/guest-bookwormos.tar

echo
echo "Building the guestos"
sudo cml_build_guestos build guest-bookworm

echo
echo "Moving guest OS to '/var/lib/cml/operatingsystems'"
sudo cp -r out/gyroidos-guests/* /var/lib/cml/operatingsystems

echo
echo "Disabling signed configs for this example"
echo "signed_configs: false" >> /etc/cml/device.conf

echo
echo "Restarting cmld service"
sudo systemctl stop cmld.service
sudo systemctl start cmld.service
echo "Verifying that restart was successful"
systemctl is-active --quiet cmld.service

echo
echo "Verifying that the guest os was successfully registred"
cml-control list_guestos | grep -q "guest-bookworm"

echo
echo "Creating GyroidOS container"
cml-control create conf/guest-bookwormcontainer.conf

echo
echo "Updating container password to be empty"
printf "trustme\n\n\n" | cml-control change_pin "guest-bookwormcontainer"

echo
echo "Starting the container"
while true
do
    if $(printf "\n" | cml-control start "guest-bookwormcontainer" | grep -q "CONTAINER_START_OK"); then
        echo "Container started successfully"
        break
    fi
    
    echo "Container did not start successfully. Waiting for 2 seconds before retrying"
    sleep 2
done

echo
echo "Waiting for the container to be ready"
while true
do
    if $(cml-control list "guest-bookwormcontainer" | grep -q "state: RUNNING"); then
        echo "Container is running"
        break
    else if $(cml-control list "guest-bookwormcontainer" | grep -q "state: STOPPED"); then
        echo "Container has stopped. Some error has occurred."
        exit 1
    fi
    
    echo "Container is not yet running. Waiting for 2 seconds before retrying"
    sleep 2
done

echo
echo "Determening container's PID"
CONTAINER_PID=$(ps auxf | grep -A 10 "[/]usr/sbin/cmld" | grep /sbin/init | awk '{print $2}')

echo "To connect to the container run:"
echo "sudo nsenter -at $CONTAINER_PID"
</pre>
</details>

## Old
2. Install your guestos as described in [GuestOS configuration](/operate/guestos_config)
6. You can now use control as described in [Basic Operation](/operate/control)
