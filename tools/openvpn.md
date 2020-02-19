# OpenVPN

Note that all following recommendations and steps works for default configurations of the respective Operating Systems, if you are using a custom DNS configurations it might cause troubles.

## Setup OpenVPN for Windows
We recommend using the official OpenVPN Community client: https://openvpn.net/community-downloads/

## Setup OpenVPN for macOS 
We recommend using https://tunnelblick.net/

## Setup OpenVPN for Ubuntu

Install the needed packages and scripts
```
sudo apt-get install -y openresolv openvpn-systemd-resolved 
sudo mkdir -p /etc/openvpn/scripts
sudo wget https://raw.githubusercontent.com/alfredopalhares/openvpn-update-resolv-conf/master/update-resolv-conf.sh -P /etc/openvpn/scripts/
sudo chmod +x /etc/openvpn/scripts/update-systemd-resolved.sh
```

Edit your `.ovpn` file by adding the following under the `client` section:
```
script-security 2
up /etc/openvpn/scripts/update-resolv-conf.sh
down /etc/openvpn/scripts/update-resolv-conf.sh
```

Now to launch OpenVPN run:
```
sudo openvpn --config /path/to/ovpn/file
```

### Troubleshooting DNS resolving

If DNS resolving for the resources behind the VPN is not working correctly try the following steps, after each step restart the openvpn connection and test if DNS works. You might not need all or any of the steps, depending on the state of your current system configurations.

This guide is tested on Ubuntu 18.04 LTS, we will keep adding troubleshooting steps whenever we encounter them to make sure it covers more cases. Do not hesitate to let us know if it did not work for you or if you have any additions.

- check the content of `/etc/nsswitch.conf`, make sure `dns` is not included. i.e, instead of:
```
hosts:          files  mdns4_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] dns myhostname
```
it should be:
```
hosts:          files dns mdns4_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] myhostname
```

- check `/etc/systemd/resolved.conf`, if dns is enabled, make sure to comment it out and run:
```
systemctl restart systemd-resolved.service
```

- if `/etc/resolv.conf` is a symlink try to remove it and restart openvon. Check with `ls -l /etc/resolv.conf`, if it turns to be a symlink remove it by running `sudo rm /etc/resolv.conf`


