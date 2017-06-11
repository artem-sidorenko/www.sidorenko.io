+++
date = "2012-01-29T13:35:42+01:00"
tags = [ "linux", "security", "apache", "otp" ]
title = "Using One-Time-Password with Apache"
+++

We are going to implement authentication with One-Time-Passwords generated via the HOTP/OATP algoritm([see RFC](http://www.ietf.org/rfc/rfc4226.txt)) with Apache. We will use [mod_authn_otp](http://code.google.com/p/mod-authn-otp/) for it.

This example is for Gentoo.

<!-- more -->

# Installation

Download and install mod_authn_otp. Ebuild for gentoo is available in [my overlay](https://github.com/artem-sidorenko/portage-2realities).

Enable it in apache configuration

```text
LoadModule authn_otp_module modules/mod_authn_otp.so`
```

Gentoo way: add "-D AUTHN_OTP" in /etc/conf.d/apache2

# Tokens

You can get a hardware token(really cheap 10-20€), list with some of them is available [here](http://code.google.com/p/mod-authn-otp/wiki/Tokens). Otherwise it's also possible to use software tokens for example on smartphones, for example [Android Token](http://code.google.com/p/androidtoken/).

# Server configuration

## Creating OTP users file

```bash
cd /etc/apache2
mkdir otp
chown apache:apache otp
```

I agree, it's not really secure to let apache create files in this directory, but it's [required](http://code.google.com/p/mod-authn-otp/wiki/Configuration#OTPAuthUsersFile) by mod_authn_otp.

Place otp.users in this directory

```text
#Token Types:

#       HOTP            - HOTP event-based token with six digit OTP
#       HOTP/E          - HOTP event-based token with six digit OTP

#       HOTP/E/8        - HOTP event-based token with eight digit OTP
#       HOTP/T30        - HOTP time-based token with 30 second interval and six digit OTP

#       HOTP/T60        - HOTP time-based token with 60 second interval and six digit OTP
#       HOTP/T60/5      - HOTP time-based token with 60 second interval and five digit OTP

#       MOTP            - Mobile-OTP time-based token 10 second interval and six digit OTP
#       MOTP/E          - Mobile-OTP event-based token with six digit OTP

#Type Username PIN Seed

#we are using time-based token with 30 seconds and our user has no PIN.
HOTP/T30 user - bfdc1e7020e88dfaa4785136156929020258121d
```

If you are using PIN, you have to prefix your token with this PIN

Change the permissions

```bash
chown root:apache otp.users
chmod 660 otp.users
```

## Authentication configuration with Apache

Create authentication configuration like here

```text
<Directory "/protected/stuff">`
    AuthType basic
    AuthName "My Protected Area"
    AuthBasicProvider OTP
    Require valid-user
    OTPAuthUsersFile /etc/apache2/otp/otp.users
    OTPAuthLogoutOnIPChange On
    OTPAuthMaxLinger 600
</Directory>
```

# Known issues

- Problems with time-based software tokens because clock offset on the mobile. Workaround is to use [ClockSync](https///market.android.com/details?id=ru.org.amip.ClockSync) on Android
