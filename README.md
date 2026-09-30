# lima_templates

## Requirements
* macOS / Linux
  * `vz` and `virtiofs` require macOS 13+ or Linux
  * Working options for macOS 12 and below are `qemu` and `9p`
* [lima](https://lima-vm.io/docs/installation/)

## Notes
* `fedora-kde.yml` uses `networks: [{vzNAT: true}]`, so it needs no `socket_vmnet`
  setup and ignores `_config/networks.yaml`. Its desktop window is a fixed
  1920x1200, software-rendered (no GPU acceleration) scanout with no clipboard
  sharing between host and guest.

## Quickstart
```bash
# install (macOS)
brew install lima socket_vmnet

# install (linux)
version=$(curl -fsSL https://api.github.com/repos/lima-vm/lima/releases/latest | jq -r .tag_name)
curl -fsSL "https://github.com/lima-vm/lima/releases/download/${version}/lima-${version:1}-$(uname -s)-$(uname -m).tar.gz" | tar Cxzvm /usr/local

# copy the network configuration
cp _config/networks.yaml ~/.lima/_config/

# setup the sudoers file for launching socket_vmnet from lima
limactl sudoers > /tmp/etc_sudoers.d_lima
sudo install -o root /tmp/etc_sudoers.d_lima /etc/sudoers.d/lima

# create alias
echo "alias lc='limactl'" >> ~/.bashrc

# create a new instance
lc start ubuntu-2004.yml --tty=false

# create a KDE Plasma desktop instance (vz driver, opens in a native window)
lc start fedora-kde.yml --tty=false

# stop the instance
lc stop ubuntu-2004

# start the instance
lc start ubuntu-2004

# remove the instance
lc rm ubuntu-2004

# list all instances
lc ls

# shell into the instance
lc shell ubuntu-2004
```
