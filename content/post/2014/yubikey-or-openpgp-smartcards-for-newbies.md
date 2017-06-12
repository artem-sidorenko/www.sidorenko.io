+++
date = "2014-11-04T20:19:17+01:00"
tags = [ "security", "ssh", "ubuntu", "mint", "smartcard" ]
title = "Yubikey or OpenPGP smartcards for newbies"
+++

I got a brand new [yubikey neo][Yubikey Neo] and wanted to get it running on my Mint 17 MATE(based on Ubuntu 14.04 Trusty Tahr) installation for GPG encryption and SSH authentification. It turned out to be not an well-transparent and easy task.
So this post gives my expirience on this topic, but isn't limited to [Yubikey][Yubikey Neo] only and should apply to other [OpenPGP cards][openpgp card] as well.

<!--more-->

# Introduction

The [OpenPGP card][openpgp wiki] is an smart card implemtation, which is supported by [GnuPG/GPG][gpg] and supports all required tasks like encryption, decryption, signing/verification, authentification.
Yubikey Neo implements OpenPGP card support (besides other interesting features of Yubikey, like OTP, upcoming [U2F](https://www.yubico.com/applications/fido/)) via [accoring Java applet][yubikey openpgp], which is usually installed on the key or [can be manually installed as well](https://github.com/Yubico/ykneo-openpgp/blob/master/doc/InstallCAPFile.txt).
To be able to use this OpenPGP functionality you have to switch the CCID (Chip Card Inteface Driver) mode of your Yubikey with a [neo manager(neoman)][yubikey neoman] or even [ykpersonalize CLI tool][yubikey personalization]. When this step is done, your Yubikey looks like a usual OpenPGP smartcard and should be recognized by according tools.

# Problems and key points

- You should use GnuPG version 2 (its a requierement for some of OpenPGP cards)
- If you don't have pcscd daemon running, neoman will not recognize the key in CCID only mode. ([bug report](https://github.com/Yubico/yubikey-neo-manager-dpkg/issues/2))
- If you want to use the smartcard for ssh authentification, you need to do it via gpg-agent with an enabled ssh-agent-support (it works well)
- Most of OpenPGP cards support up to 2048 bit keys ([incl. yubikey](https://github.com/Yubico/ykneo-openpgp/issues/2)), [OpenPGP SmartCard v2][openpgp card] support up to 4096 bit keys
- Provisioning of smartcard is done by gpg and is a quite straight forward process
- Desktop environments like MATE or KDE have usually own keyring service, which implements gpg/ssh-agent functionality. However gnome-keyring (used in MATE) [doesn't support smartcards](https://bugs.launchpad.net/ubuntu/+source/gnome-keyring/+bug/884856) and your `gpg --card-status` or `gpg --card-edit` commands will fail

```bash
$ gpg --card-status
gpg: selecting openpgp failed: unknown command
gpg: OpenPGP card not available: general error
```

- Its not really transparent (gconf configuration doesn't work anymore) how to disable the gpg/ssh agents of gnome-keyring (you probably still want to keep it for secrets of other applications, e.g. NetworkManager). Gnome-keyring is started by [session-manager without any further parameters][mate-session keyring start] and uses its [precompiled defaults][gnome-keyring defaults] to bring up the services (usually all services incl. ssh and gpg agents).

# Lets go

- On Ubuntu/Mint you will need the [yubico ppa][yubico ppa] if you want the Yubico management tools

```bash
$ sudo apt-add-repository ppa:yubico/stable
$ sudo apt-get update
$ sudo apt-get install yubikey-personalization-gui yubikey-neo-manager yubikey-personalization
```

- pcscd, pcsc-tools, scdaemon and gpg2 are required as well

```bash
$ sudo apt-get install pcscd scdaemon gnupg2 pcsc-tools
```

- If you want to enfore usage of gpg2, create a symlink

```bash
$ ln -s /usr/bin/gpg2 /usr/local/bin/gpg
```

- If you use gnome-keyring, place the following file to `/usr/local/bin/gnome-keyring-daemon` and do it executable, its a workaround for issue described above. Check your $PATH, if everything is fine, this file will be used by mate-session to start the gnome-keyring

```bash
#!/bin/sh
# /usr/local/bin/gnome-keyring-daemon

/usr/bin/gnome-keyring-daemon --start -c pkcs11,secrets
```

- Enable GnuPG agents

```bash
echo "use-agent" >> ~/.gnupg/gpg.conf
echo "enable-ssh-support" >> ~/.gnupg/gpg-agent.conf
```

- If you have multiple readers (I've one for real smartcards and yubikey is another one) you can specify which reader should be used

```bash
$ pcsc_scan -n
...
Scanning present readers...
0: Gemalto GemPC Express 00 00
1: Yubico Yubikey NEO CCID 01 00
...
#Ctrl+C
$ echo "reader-port \"Yubico Yubikey NEO CCID 01 00\"" > ~/.gnupg/scdaemon.conf
```

- In case of multiple readers (espicially with different devices) or some issues with hanging scdaemon, you can create `~/.gnupg/scd-event` script (and make it executable) which will kill scdaemon when the smartcard is pulled out

```bash
#!/bin/sh
# ~/.gnupg/scd-event

state=$8

if [ "$state" = "NOCARD" ]; then
  pkill -9 scdaemon
fi
```

- Relogin (GnuPG agents will be started by /etc/X11/Xsession.d/90gpg-agent on Ubuntu/Mint)

# Creation of keys

- First of all enable CCID mode of Yubikey with neoman
- We will create a new set of keys with according subkeys ([more details on subkeys][gpg manual keymgnt])
- The private master key will be stored offline on a usb stick, private subkeys on the smartcard
- The master key will be used only for signing (of subkeys etc) and subkeys on the smartcard for everything else
- The expire date will be 1 year, with manual extending every year ([see here the reasons why][openpgp expiration reasons])

```bash
#Adapted from http://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/
#Assuming your usb stick is mounted at /media/user/stick
# gpghome is for gnupg keyrings and armor-baks for text backups
$ mkdir /media/user/stick/gpghome /media/user/stick/armor-baks
$ chmod 700 /media/user/stick/gpghome /media/user/stick/armor-baks
$ export GNUPGHOME=/media/user/stick/gpghome
# crypto settings
$ cat <<EOF > $GNUPGHOME/gpg.conf
default-preference-list SHA512 SHA384 SHA256
cert-digest-algo SHA512
use-agent
EOF

#generate a new master key
$ gpg --gen-key
...
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
...
Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
# create revocation certificate
$ gpg --output $GNUPGHOME/../armor-baks/revocation-cert --gen-revoke <<key id here>>
...
Please select the reason for the revocation:
  0 = No reason specified
  1 = Key has been compromised
  2 = Key is superseded
  3 = Key is no longer used
  Q = Cancel
(Probably you want to select 1 here)
Your decision? 1
Enter an optional description; end it with an empty line:
> Created during key creation, just for an emergency case
>
Reason for revocation: Key has been compromised
Created during key creation, just for an emergency case
Is this okay? (y/N) y
...
# do a text backup of master key
$ gpg -a --export-secret-keys > $GNUPGHOME/../armor-baks/masterkeys.txt
# Create subkeys: signing, encryption, authentification (for ssh)
$ gpg --expert --edit-key <<key id here>>
...
gpg> addkey
...
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
...
Key is valid for? (0) 1y
...
gpg> addkey
...
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
...
Key is valid for? (0) 1y
...
gpg> addkey
...
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt
...
Your selection? S

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt
...
Your selection? E

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions:
...
Your selection? A

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate
...
Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
Requested keysize is 2048 bits
Please specify how long the key should be valid.
...
Key is valid for? (0) 1y
...
gpg> save
# backup subkeys
$ gpg -a --export-secret-subkeys > $GNUPGHOME/../armor-baks/subkeys.txt
# configure your smartcard
$ gpg --card-edit
...
gpg/card> admin
...
1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit
...
# Keep in mind, PIN should be >=6 digits and Admin PIN >= 8 digits
Your selection? 3
PIN changed.
...
Your selection? q
gpg/card> name
...
gpg/card> lang
...
gpg/card> url
...
gpg/card> sex
...
gpg/card> login
...
gpg/card> quit
# move subkeys to the smartcard: signature, encryption, authentification
$ gpg --edit-key <<key id here>>
...
gpg> toggle
...
gpg> key 1
...
gpg> keytocard
...
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
...
gpg> key 1
...
gpg> key 2
...
gpg> keytocard
...
Please select where to store the key:
   (2) Encryption key
Your selection? 2
...
gpg> key 2
...
gpg> key 3
...
gpg> keytocard
...
Please select where to store the key:
   (3) Authentication key
Your selection? 3
...
gpg> save
# do another backup
$ gpg -a --export-secret-keys > $GNUPGHOME/../armor-baks/masterkeys-stubs.txt
$ gpg -a --export-secret-subkeys > $GNUPGHOME/../armor-baks/subkeys-stubs.txt
$ gpg -a --export > $GNUPGHOME/../armor-baks/publickey.txt
```

- Transfer the ``publickey.txt`` to the systems you want to use smartcard on

```bash
# import
$ gpg --import < publickey.txt
# let gpg see the smartcard and generate secret stubs
$ gpg --card-status
# configure trust
$ gpg --edit-key <<key id here>>
...
gpg> trust
...
  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
...
gpg> quit
```

- Thats all, you can see your authentification ssh public key via ``ssh-add -L``

# Migration of old SSH keys

- Copy your keys to some other place (name of file will be a key comment used by gnupg ssh agent)
- Just ssh-add them, they will be imported and stored in ``~/.gnupg/private-keys-v1.d/``

```bash
$ mkdir /tmp/keys
$ cp -p ~/.ssh/id_rsa /tmp/keys/my\ very\ cool\ old\ key
$ cd /tmp/keys
$ ssh-add my\ very\ cool\ old\ key
```

- You will be asked for a current and a new password for GPG key store
- If you want to see the public keys, just use ``ssh-add -L``

# See too

- [Functional Specification of the OpenPGP application on ISO Smart Card Operating Systems](http://g10code.com/docs/openpgp-card-2.0.pdf)
- [GnuPG SmartCard Howto][gpg smartcard howto]
- [Use OpenPGP Keys for OpenSSH, how to use gpg with ssh](https://www.programmierecke.net/howto/gpg-ssh.html)
- [SSH Authentification with your PGP key](http://budts.be/weblog/2012/08/ssh-authentication-with-your-pgp-key)
- [The GNU Privacy Handbook Key Management][gpg manual keymgnt]
- [Offline GnuPG Master Key and Subkeys on YubiKey NEO Smartcard](http://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/)

[yubikey neo]: https://www.yubico.com/products/yubikey-hardware/yubikey-neo/
[openpgp card]: http://shop.kernelconcepts.de/product_info.php?cPath=1_26&products_id=42
[openpgp wiki]: https://en.wikipedia.org/wiki/OpenPGP_card
[gpg]: https://www.gnupg.org
[gpg smartcard howto]: https://www.gnupg.org/howtos/card-howto/en/smartcard-howto.html
[yubikey openpgp]: https://github.com/Yubico/ykneo-openpgp
[yubikey manual]: https://www.yubico.com/wp-content/uploads/2014/10/YubiKey-Manual-v3.3.pdf
[yubikey neoman]: https://github.com/Yubico/yubikey-neo-manager
[yubikey personalization]: https://github.com/Yubico/yubikey-personalization
[yubico ppa]: https://launchpad.net/~yubico/+archive/ubuntu/stable
[gnome-keyring defaults]: https://git.gnome.org/browse/gnome-keyring/tree/daemon/gkd-main.c#n89
[mate-session keyring start]: https://github.com/mate-desktop/mate-session-manager/blob/master/mate-session/msm-gnome.c#L94-L100
[gpg manual keymgnt]: https://www.gnupg.org/gph/en/manual.html#MANAGEMENT
[openpgp expiration reasons]: http://blog.josefsson.org/2014/08/26/the-case-for-short-openpgp-key-validity-periods/
