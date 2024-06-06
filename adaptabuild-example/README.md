adaptabuild-example

A workspace that is a kitchen-sink for all the targets and modules we
might need to do embedded devlopment

Useful Links:

vcan for WSL2 kernel ... https://unix.stackexchange.com/a/702288

https://blog.beamconnectivity.com/can-bus-support-in-windows-11-84520ff1fbd3
https://github.com/Beam-Connectivity/beam-labs/tree/main/vcan-in-wsl2



# #!/bin/bash

# Exit if an error occurs
set -e

# Get the directory of the script
SCRIPT_PATH="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# Update and install required packages
echo !!!!!!!!!!!!!!!!!!!!!!!!!
echo Install required packages
echo !!!!!!!!!!!!!!!!!!!!!!!!!
sudo apt update -y
sudo apt install -y --no-install-recommends libelf-dev flex bison libssl-dev libncurses-dev bc build-essential make
sudo apt install -y --no-install-recommends wslu

# The v1.21 of dwarves (and its dependency pahole) is required.
echo !!!!!!!!!!!!!!!!!!!!!
echo Install dwarves v1.21
echo !!!!!!!!!!!!!!!!!!!!!
if [[ ! -f "dwarves_1.21-0ubuntu1~20.04.1_amd64.deb" ]]; then
echo !!!!!!!!!!!!!!Downloading dwarves 1.21!!!!!!!!!!!!!!!!!!!!!
wget http://archive.ubuntu.com/ubuntu/pool/universe/d/dwarves-dfsg/dwarves_1.21-0ubuntu1~20.04.1_amd64.deb
fi
sudo apt install ./dwarves_1.21-0ubuntu1~20.04.1_amd64.deb

# Clone and checkout the WSL2 Linux Kernel
echo !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
echo Clone and checkout the WSL2 Linux Kernel
echo !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
if [[ ! -d "./WSL2-Linux-Kernel/" ]]; then
    git clone https://github.com/microsoft/WSL2-Linux-Kernel --shallow-since=2022-07-30
fi
cd WSL2-Linux-Kernel/
git checkout linux-msft-wsl-5.15.57.1 # This version has canfd drivers as well as regular can drivers
cat /proc/config.gz | gunzip > .config
make prepare modules_prepare

# Configure kernel
echo !!!!!!!!!!!!!!!!
echo Configure kernel
echo !!!!!!!!!!!!!!!!
make menuconfig 

# Build the kernel
echo !!!!!!!!!!!!
echo Build kernel
echo !!!!!!!!!!!!
make -j$(( $(nproc) - 1 ))
sudo make modules_install
sudo ln -s /lib/modules/5.15.57.1-microsoft-standard-WSL2+/ /lib/modules/5.15.57.1-microsoft-standard-WSL2

# Find and store Windows username
WIN_USER=$(wslpath "$(wslvar USERPROFILE)")

# Copy kernel to Windows home directory
echo !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
echo Copy kernel and wsl-reboot.sh to Windows
echo !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
cp vmlinux $WIN_USER
cd $WIN_USER
touch .wslconfig
cd $SCRIPT_PATH
cp wsl-reboot.ps1 $WIN_USER
echo Done! Now close this terminal, open a Windows powershell and launch the wsl-reboot.ps1 script to reboot WSL with the custom linux kernel.