+++
date = "2014-10-15T13:36:39+01:00"
tags = [ "linux", "network", "telekom" ]
title = "Telekom Hotspot autologin with NetworkManager"
+++

It was always annoying for me to type in the login credentials on the Hotspots of Deutsche Telekom. As I'm lazy, here is a script which can be integrated with networkmanager via dispatcher.d interface, which checks for the right interface and right SSID, then calls the login page with according credentials.

<!--more-->

- Put [this script](https://raw.githubusercontent.com/artem-sidorenko/scripts/master/networkmanager-telekom-hotspot/02hotspot) to ``/etc/NetworkManager/dispatcher.d/02hotspot``
- Put the [according settings file](https://raw.githubusercontent.com/artem-sidorenko/scripts/master/networkmanager-telekom-hotspot/hotspot-config) to ``/etc/NetworkManager/hotspot-config``

- update the username and password in the config file
- depending on the distribution you might have to change the wlan interface name
- if you want to automatically to connect to the VPN, configure it. If your VPN certificate is password protected, you have to update your connection like this:
  - open the file ``/etc/NetworkManager/system-connections/<<VPNNAME>>``
  - set ``cert-pass-flags`` in the vpn section to 0
  - add a section ``vpn-secrets`` with an option ``cert-pass`` with a password as a value

```ini
...
[vpn]
...
cert-pass-flags=0
...

[vpn-secrets]
cert-pass=<<certificate password here>>
...
```

- update the permissions of the file ``chmod 0700 /etc/NetworkManager/hotspot-config``

# Known limitations

- Currently only Telekom Hotspot is supported, support of ICE and probably some other Hotspots can be added easy

# See too

- [Automatic Wi-Fi Hotspot Login and VPN Activation Script](http://mobilesociety.typepad.com/mobile_life/2010/08/automatic-wifi-hotspot-login-and-vpn-activation-script.html)
- [Auto weblogin](https://github.com/arlimus/weblogin)
- [Error when trying to connect to VPN on startup](http://askubuntu.com/questions/198136/error-when-trying-to-connect-to-vpn-on-startup)
