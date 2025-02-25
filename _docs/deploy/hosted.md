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
apt-get install git build-essential unzip re2c pkg-config check \
    lxcfs libprotobuf-c-dev automake libtool libselinux1-dev \
    libcap-dev protobuf-c-compiler libssl-dev udhcpd udhcpd libsystemd-dev
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
1. Run `cml-scd` once to initalize `/var/lib/cml/tokens`: `sudo cml-scd`
2. Create `cml-control` group: `sudo addgroup cml-control`
3. Add current user to the group: `sudo usermod -aG cml-control $(whoami)`
4. Create test certificates with `cml_gen_dev_certs test-certs`. Note that the `test-certs` directory should not exist.
5. Install root certificate: `sudo cp test-certs/ssig_rootca.cert /var/lib/cml/tokens/`
6. Start `cml-scd.service` with `sudo systemctl start cml-scd.service`

3. You can either push your own certificate by using cml-control push_ca or copy the ssig_rootca.cert from your build to /var/lib/cml/tokens
4. Run scd again, this time it will start a loop and keep running
5. Run cmld and keep it running
2. Install your guestos as described in [GuestOS configuration](/operate/guestos_config)
6. You can now use control as described in [Basic Operation](/operate/control)

