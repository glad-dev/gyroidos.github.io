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

### Automatic

Download and run the [guest setup script](/assets/hosted-debian-guest.sh), which will create and start a debian 12 container.

### Manually

1. Create a new folder for the guest os
2. Initalize it with `cml_build_guestos init $GUEST_NAME --pki /path/to/cml/certs`
3. Create a new rootfs, e.g. using `debootstrap`
4. Add it to an uncompressed tarball called `${GUEST_NAME}os.tar`
5. Move it the `rootfs/` directory created in step 2
6. Build the guest os with `cml_build_guestos build $GUEST_NAME`
7. Add the guest os to cml by copying the content of `out/` to `/var/lib/cml/operatingsystems`
8. For this example, append `signed_configs: false` to `/etc/cml/device.conf`
9. Restart the `cmld` service
10. Verify that the guestos was detected by running `cml-control list_guestos`, which should contains the new guest os
11. Create the GyroidOS container with `cml-control create conf/${GUEST_NAME}container.conf`
12. Change the container's password with `cml-control change_pin "$GUEST_NAME"`. Default password is "trustme"
13. Start the container with `cml-control start "$GUEST_NAME"`. If command returns `CONTAINER_START_EINTERNAL`, run it again.
14. Verify that it is running with `cml-control list "$GUEST_NAME"`, which should have `state: RUNNING`
15. To connect to the container, run `ps auxf`, search for `/user/sbin/cmld`. It should have a child process running `/sbin/init`. Get the PID and connect with `sudo nsenter -at $PID`

For more details, see the [GuestOS configuration](/operate/guestos_config) and the [Basic Operation](/operate/control) documentation pages.
