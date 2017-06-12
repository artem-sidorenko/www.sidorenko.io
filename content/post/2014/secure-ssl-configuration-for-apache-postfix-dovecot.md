+++
date = "2014-02-13T13:50:13+01:00"
tags = [ "security", "apache", "postfix", "dovecot", "ssl" ]
title = "Secure SSL configuration for apache, postfix, dovecot"
+++

There are already several articles about perfect forward secrecy and safe ssl configuration with according recommendations on the net, like this

- [Configuring apache, nginx][qualys_apache_guide]
- [German: Postfix TLS perfect forward secrecy][sys4_postfix_guide]
- [German: Perfect Forward Secrecy (PFS) für Postfix und Dovecot einrichten][heinlein_postfix_dovecot_guide]
- [German: Dovecot TLS Perfect Forward Secrecy][sys4_dovecot_guide]
- [Applied Crypto Hardening][bettercrypto]

But I missed somehow a short overview for me with verification instructions and all information links in one place. So this article is going to cover Perfect Forward Secrecy(PFS) for the software: apache, postfix, dovecot and represents somehow a summary over different information.

<!--more-->

# Apache

- ensure you have Apache >=2.2.26. Apache versions [below doesn't support][apache_changelog] [ECC][wikipedia_ecc] and [ECDH][wikipedia_ecdh] which are used for Forward Secrecy implementation.
- You should have openssl >= 1.0.0
- Switch everything to HTTPS only, avoid any mixed content. We will enforce the  [HSTS][wikipedia_hsts] policy
- Add following things to your SSL configuration within apache, we keep RC4 as the last option and SSLv3 to have compatibility to the Windows XP clients.

```text
#don't allow SSLv2 and SSLv3, its unsecure
SSLProtocol all -SSLv2 -SSLv3
#enforce the server cipher preference, don't trust the client config
SSLHonorCipherOrder on
#allowed cipher suites
SSLCipherSuite "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 \
EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 \
EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4"
#don't use sslcompression, its unsecure
SSLCompression off
#Enable HTTP strict transport security policy(HSTS):
# don't allow any unencrypted communication here and on the subdomains
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

- Restart apache and test your configuration

# Postfix

- you should have [openssl >=1.0.0 and postfix >=2.6][postfix_pfs]
- generate DH params, we don't go with 2048-bit EDH as not all clients might support this

```bash
$ openssl gendh -out /etc/postfix/dh_512.pem -2 512
$ openssl gendh -out /etc/postfix/dh_1024.pem -2 1024
```

- update the ssl settings in the main.cf
  - we don't change the smtp client configuration to avoid any problems with outband mails. Target system has to provide the prefered encryption, not us
  - in my configuration smtpd_tls_security_level=encrypt is configured for port 587 as MSA for client submission and smtpd_tls_security_level=may is configured for port 25 as MTA for accepting inbound mails. We don't disallow any ciphers for MTA, as the alternative is plain-text. If you want to enforce strong ciphers uncomment the lines with smtpd_tls_security_level=may

```text
#the dh params
smtpd_tls_dh1024_param_file = /etc/postfix/dh_1024.pem
smtpd_tls_dh512_param_file = /etc/postfix/dh_512.pem
#enable ECDH
smtpd_tls_eecdh_grade = strong
#enabled SSL protocols, don't allow SSLv2 and SSLv3
smtpd_tls_protocols= !SSLv2, !SSLv3
smtpd_tls_mandatory_protocols= !SSLv2, !SSLv3
#allowed ciphers for smtpd_tls_security_level=encrypt
smtpd_tls_mandatory_ciphers = high
#allowed ciphers for smtpd_tls_security_level=may
#smtpd_tls_ciphers = high
#enforce the server cipher preference
tls_preempt_cipherlist = yes
#disable following ciphers for smtpd_tls_security_level=encrypt
smtpd_tls_mandatory_exclude_ciphers = aNULL, MD5 , DES, ADH, RC4, PSD, SRP, 3DES, eNULL
#disable following ciphers for smtpd_tls_security_level=may
#smtpd_tls_exclude_ciphers = aNULL, MD5 , DES, ADH, RC4, PSD, SRP, 3DES, eNULL
#enable TLS logging to see the ciphers for inbound connections
smtpd_tls_loglevel = 1
#enable TLS logging to see the ciphers for outbound connections
smtp_tls_loglevel = 1
```

- restart postfix and test your configuration
- this configuration works for me for Thunderbird and Kaiten Mail/K9-Mail without problems

# Dovecot

- you should have openssl >=1.0.0 dovecot >=2.1.x required, better dovecot >=2.2.x because of ECDHE support
- Dovecot tryies to use PFS by default, so besides the enabled SSL almost no actions are required
- change the log settings to see the cipher, grep for a login_log_format_elements in dovecot configs and add %k to it

```text
login_log_format_elements = "user=<%u> method=%m rip=%r lip=%l mpid=%e %c %k"
```

- configure the allowed ciphers. Server side enforcement works only for [dovecot  >=2.2.6][dovecot_enforcement]

```text
ssl_cipher_list = EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH+aRSA+RC4:EECDH:EDH+aRSA:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS:!RC4
#only for dovecot >=2.2.6, enforce the server cipher preference
ssl_prefer_server_ciphers = yes
#disable SSLv2 and SSLv3
ssl_protocols = !SSLv2 !SSLv3
```

- restart dovecot and test the configuration
- this configuration works for me for Thunderbird and Kaiten Mail/K9-Mail without problems

# Testing

You can verify the configuration via different ways

- for https you can use the [great SSLLabs test site][ssllabs_ssltest]
- you can test it with openssl client to verify the specific protocol/cipher

```bash
#a help page with all possible options
$ openssl s_client --help
#https
$ openssl s_client -connect [server]:443
#try SSLv2 with https which shouldn't work
$ openssl s_client -connect [server]:443 -ssl2
#smtp with starttls
$ openssl s_client -starttls smtp -connect [server]:25
#imap
$ openssl s_client -starttls imap -connect [server]:143
```

- I found a [script][ssl_scan_bash] which can be used for a entire scan
- you can use the [sslscan][sslscan] for https and smtp to scan all options (only smtp is supported with starttls)

```bash
#test https
$ sslscan [server]:443
#test smtp with starttls
$ sslscan --starttls [server]:25
```

- I didn't try it, but you can take a look to a [fork of sslscan][sslscan_fork] or [sslyze][sslyze] which should support imap with startssl

**Updated on 18.11.2014 with SSLv3 due to Poodle**

# See too

- [SSLLabs SSL Test][ssllabs_ssltest] and [SSL Server Rating Guide][qualys_rating]
- [SSL/TLS Deployment Best Practices][qualys_deployment]
- [SSL Labs: Deploying Forward Secrecy][ssllabs_deploying_pfs]
- [OpenSSL Cookbook][openssl_cookbook]
- [Configuring apache, nginx, openssl][qualys_apache_guide]
- [German: Postfix TLS perfect forward secrecy][sys4_postfix_guide]
- [German: Perfect Forward Secrecy (PFS) für Postfix und Dovecot einrichten][heinlein_postfix_dovecot_guide]
- [German: Dovecot TLS Perfect Forward Secrecy][sys4_dovecot_guide]
- [Applied Crypto Hardening][bettercrypto]
- [TLS Forward Secrecy in Postfix][postfix_pfs]
- [Perfect Forward Secrecy mit Apple Mail?][kuketz_dovecot]
- [CryptoNark SSL scanner][cryptonark]


[wikipedia_ecc]: http://en.wikipedia.org/wiki/Elliptic_Curve_Cryptography
[wikipedia_ecdh]: http://en.wikipedia.org/wiki/Elliptic_curve_Diffie%E2%80%93Hellman
[apache_changelog]: http://www.apache.org/dist/httpd/CHANGES_2.2.26
[ssllabs_ssltest]: https://www.ssllabs.com/ssltest/
[qualys_apache_guide]: https://community.qualys.com/blogs/securitylabs/2013/08/05/configuring-apache-nginx-and-openssl-for-forward-secrecy
[sys4_postfix_guide]: http://sys4.de/de/blog/2013/08/14/postfix-tls-forward-secrecy/
[heinlein_postfix_dovecot_guide]: http://www.heinlein-support.de/blog/security/perfect-forward-secrecy-pfs-fur-postfix-und-dovecot/
[wikipedia_hsts]: http://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security
[sslscan]: http://sourceforge.net/projects/sslscan/
[sslscan_fork]: https://github.com/ioerror/sslscan
[ssl_scan_bash]: http://superuser.com/questions/109213/is-there-a-tool-that-can-test-what-ssl-tls-cipher-suites-a-particular-website-of
[postfix_pfs]: http://www.postfix.org/FORWARD_SECRECY_README.html
[dovecot_enforcement]: http://www.dovecot.org/list/dovecot-news/2013-September/000262.html
[sys4_dovecot_guide]: https://sys4.de/de/blog/2013/08/15/dovecot-tls-perfect-forward-secrecy/
[bettercrypto]: https://bettercrypto.org/static/applied-crypto-hardening.pdf
[sslyze]: https://github.com/iSECPartners/sslyze
[cryptonark]: http://blog.techstacks.com/cryptonark.html
[qualys_deployment]: https://www.ssllabs.com/projects/best-practices/index.html
[qualys_rating]: http://www.ssllabs.com/projects/rating-guide/index.html
[ssllabs_deploying_pfs]: https://community.qualys.com/blogs/securitylabs/2013/06/25/ssl-labs-deploying-forward-secrecy
[openssl_cookbook]: https://www.feistyduck.com/books/openssl-cookbook/
[kuketz_dovecot]: http://www.kuketz-blog.de/perfect-forward-secrecy-mit-apple-mail/
