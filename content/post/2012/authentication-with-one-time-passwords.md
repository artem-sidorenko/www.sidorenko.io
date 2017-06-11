+++
date = "2012-01-29T13:19:04+01:00"
tags = [ "linux", "security", "pam", "otp" ]
title = "Authentication with One-Time-Passwords"
+++

This article is about implementing One-Time-Password authentication with PAM, so you can use it with ssh and other pam enabled services.

We will use [oath-toolkit](http://www.nongnu.org/oath-toolkit/) for it. So install it, gentoo ebuild for it is provided in [my overlay](https://github.com/artem-sidorenko/portage-2realities).

<!-- more -->

# Creating users

Create /etc/users.oath

```text
#Token types

#       HOTP            - HOTP event-based token with six digit OTP
#       HOTP/T30        - HOTP time-based token with 30 second interval and six digit OTP

#       HOTP/T60        - HOTP time-based token with 60 second interval and six digit OTP
#type user pin seed

HOTP/T30 user - bfdc1e7020e88dfaa4785136156929020258121d
```

Change the permissions, so only root can access this file

```bash
chmod 600 /etc/users.oath
```

# Configuring PAM

Before you configure your PAM, ensure that you have at least one running root shell on this system, so you have access if something goes wrong.`

Take the configuration in required pam configuration file like this

```text
auth            sufficient      pam_oath.so usersfile=/etc/users.oath window=10 digits=6
```

Use "sufficient" if you want one to authentication methods to succeed(OTP or normal password) or use "required" or "requisite" if both have to succeed. **Dont use "sufficient" alone without any "required" after it, you will open a security hole in your system.**  More in `man pam.conf`.

Example configuration for remote login services like ssh, in /etc/pam.d/system-remote-login

```text
#here our changes, we have to define here other checks, which are usually included via system-login

#auth           include         system-login
auth            required        pam_tally2.so onerr=succeed
auth            required        pam_shells.so
auth            required        pam_nologin.so
auth            required        pam_env.so
auth            sufficient      pam_oath.so usersfile=/etc/users.oath window=10 digits=6
#comment the next line if you want to have only OTP authentication and change the previous line to required or requisite

auth            required        pam_unix.so try_first_pass likeauth nullok
auth            optional        pam_permit.so
#end of changes

account         include         system-login
password        include         system-login
session         include         system-login
```
