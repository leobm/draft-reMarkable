## Build and deploy button-capture

Change to the parent directory of the draft-reMarkable directory.
```
$ git clone https://github.com/dixonary/button-capture-reMarkable.git
$ cd button-capture-reMarkable
$ touch build.sh
$ chmod +x build.sh
```
and add following bash script to this file

```bash
#!/bin/bash
set -e
TOOLCHAIN=../draft-reMarkable/rm-toolchain
source $TOOLCHAIN/environment-setup-cortexa9hf-neon-oe-linux-gnueabi
qmake
make
set -x
```

```
$ touch button-capture.service
```
and add following configuration to this file

```
[Unit]
Description=Button Capture Listener
StartLimitIntervalSec=600
StartLimitBurst=4
After=home.mount

[Service]
ExecStart=/home/root/button-capture/button-capture
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
$ touch button-capture-install.sh
$ chmod +x button-capture-install.sh
```
and add following bash script to this file

```bash
#!/bin/bash
# This will deploy the button-capture service to the reMarkable device.
# Use at your own peril. May prompt you for password, which can be found
# in your device's standard interface. For 2.0 this is in Menu -> Settings
# -> About, scroll to the bottom.
# Source files are in $base, which by default is in the result directory.

usage() {
    echo "usage: $0 [-y] REMARKABLE_IP"
    echo
    echo "Deploys button-capture to REMARKABLE_IP. Files are installed into /home/root/button-capture, and linked to their proper
    locations."
    echo
    echo "  -y Automatic yes to all prompts (not including password prompts)"
    echo
    echo "Type '$0 --help' or '$0 -h' to see this message."
}

while getopts ":y:" option; do
    case $option in
    y)
        answer="Y"
        shift
        ;;
    *)
        usage
        exit 1
        ;;
    esac
done

if [ $# -lt 1 ] || [ $# -gt 2 ]; then
    usage
    exit 1
fi

ip_pattern='^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$'
if [[ ! "$1" =~ ${ip_pattern} ]] && [[ ! "$2" =~ ${ip_pattern} ]]; then
    usage
    exit
fi

set -e
base=.
device=$1

if [ "$answer" != "Y" ]; then
    read -p "Copy draft files to /home/root/button-capture on $device? (Y/n): " copystep
    [ $copystep != "Y" -a $copystep != 'y' ] && echo "Cancelled by user" && exit 1
fi

echo "Copying files to /home/root/button-capture"
set -x
ssh root@$device "mkdir -p /home/root/button-capture"
scp -pr $base/{button-capture,button-capture.service} root@$device:/home/root/button-capture
set +x

if [ "$answer" != "Y" ]; then
    read -p "Link files and start/stop button-capture service on $device? (Y/n): " linkstep
    [ $linkstep != "Y" -a $linkstep !='y' ] && echo "Cancelled by user" && exit 1
fi

set -x
ssh root@$device <<EOH
set -ex

cd /home/root/button-capture
cp button-capture.service /lib/systemd/system/

systemctl enable --now button-capture
EOH
```

now run the install script 

```
$ ./button-capture-install.sh <YOUR_REMARKABLE_IP>
```