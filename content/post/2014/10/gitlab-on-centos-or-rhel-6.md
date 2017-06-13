+++
date = "2014-10-29T08:40:22+01:00"
tags = [ "gitlab", "centos", "redhat" ]
title = "GitLab on Centos or RHEL 6"
+++

Gitlab provides [manual installation instructions for Ubuntu only](https://gitlab.com/gitlab-org/gitlab-ce/blob/v7.5.0/doc/install/installation.md), this post covers the procedure how to install it on CentOS/RHEL/RedHat 6.X.

<!--more-->

# Environment description

- Minimal CentOS/RHEL 6.X installation with SELinux disabled
- We will use SMTP relay without authentification for outbound mails
- We will use MySQL as DB, mysql data will be stored in ``/data/mysql``
- We will use nginx as a frontend web server
- Gitlab installation will be reachable with a gitlab prefix via https ( ``https://example.com/gitlab/``)
- Gitlab will be installed in ``/opt/gitlab`` and repositories will be in ``/data/git``

# Prepare environment

- We will use system user ``git`` with a home dir ``/opt/gitlab`` for the gitlab installation
- Because of version requierements we have to build and install own git and ruby

```bash
#create git user
$ adduser -d /opt/gitlab -m git
$ mkdir /data/git
$ chown git:git /data/git
#install build dependencies for git and ruby
$ yum -y install gcc openssl-devel libcurl-devel expat-devel perl-ExtUtils-MakeMaker gettext
#install git to /opt/git
$ cd /tmp
$ curl -L --progress https://www.kernel.org/pub/software/scm/git/git-2.1.2.tar.gz | tar xz
$ cd git-2.1.2/
$ ./configure --without-tcltk
$ make prefix=/opt/git all
$ make prefix=/opt/git install
#install ruby to /opt/ruby
$ mkdir /tmp/ruby && cd /tmp/ruby
$ curl -L --progress ftp://ftp.ruby-lang.org/pub/ruby/2.1/ruby-2.1.2.tar.gz | tar xz
$ cd ruby-2.1.2
$ ./configure --prefix=/opt/ruby --disable-install-rdoc
$ make
$ make install
$ /opt/ruby/bin/gem install bundler --no-ri --no-rdoc
#both installations should be in the PATH
$ echo 'export PATH="$PATH:/opt/ruby/bin:/opt/git/bin"' > /etc/profile.d/path.sh
$ chmod +x /etc/profile.d/path.sh
$ source /etc/profile
#install build dependencies for some ruby modules
$ yum -y install libicu-devel patch gcc-c++ cmake mysql-devel
```

# Install databases

- We will use MySql for Gitlab data (comments etc) and redis for transaction data
- Mysql data will be stored in ``/data/mysql``
- Mysql will be avaliable only via filesystem socket

```bash
#install mysql
$ yum -y install mysql-server
#create data paths
$ mkdir /data/mysql
$ chown mysql:mysql /data/mysql
#update the configuration
$ sed -i 's/datadir=\/var\/lib\/mysql/datadir=\/data\/mysql/g' /etc/my.cnf
#mysql should listen only on local sockets, no networking
$ sed -i '/\[mysqld\]/a skip-networking' /etc/my.cnf
#enable and start mysql
$ chkconfig mysqld on
$ /etc/init.d/mysqld start
#install redis
$ yum -y install redis
#disable networking, allow only filesystem socket
$ sed -i 's/^port .*/port 0/' /etc/redis.conf
$ echo 'unixsocket /var/lib/redis/redis.sock' >> /etc/redis.conf
$ echo 'unixsocketperm 770' >> /etc/redis.conf
#allow git user to connect to the socket
$ usermod -aG redis git
#enable and start redis
$ chkconfig redis on
$ /etc/init.d/redis start
```

# Install gitlab

- Gitlab will be installed under the user ``git`` in his homedirectory ``/opt/gitlab``

```bash
#first of all switch to the user git
$ sudo -u git -H -i bash
#create directory for our repos
$ mkdir /data/git/repos
#checkout gitlab 7.5.0
$ git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b v7.5.0 gitlab
$ cd gitlab
#create folder for temp repo checkouts (e.g. merges)
$ mkdir ../gitlab-satellites
$ chmod u+rwx,g=rx,o-rwx ../gitlab-satellites

#configure gitlab
$ cp config/gitlab.yml.example config/gitlab.yml
# do a global replace of our home path
$ sed -i 's/\/home\/git/\/opt\/gitlab/g' config/gitlab.yml
# edit config file
# important configuration is:
#  - email_from: dontreply@gitlab.example.com
#  - repos_path: /data/git/repos/
#  - git:
#    - bin_path: /opt/git/bin/git
#  - relative_url_root: /gitlab
#  - port: 443
#  - https: true
#  - default_can_create_group: false
$ vi config/gitlab.yml

# edit another config file
# update relative url
# - config.relative_url_root = "/gitlab"
$ vi config/application.rb

# configure smtp relay
# if you don't have to authorize yourself, just comment out the credential parameters
$ cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb
$ vi config/initializers/smtp_settings.rb

# find our the count of cpus
$ nproc
# configure gitlab web server unicorn
# important config changes are:
#  - worker_processes [here count of cpus, e.g. 1]
#  - timeout 360
#  - ENV['RAILS_RELATIVE_URL_ROOT'] = "/gitlab"
#  - comment out the line "listen "127.0.0.1:8080", :tcp_nopush => true"
$ cp config/unicorn.rb.example config/unicorn.rb
$ sed -i 's/\/home\/git/\/opt\/gitlab/g' config/unicorn.rb
$ vi config/unicorn.rb

# copy rack_attack conf
$ cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

# configure resque (used for backgroud jobs)
# important config changes are:
#  - production: unix:/var/lib/redis/redis.sock
$ cp config/resque.yml.example config/resque.yml
$ vi config/resque.yml

# configure database
# important changes are:
#  - password
#  - socket: /var/lib/mysql/mysql.sock
$ cp config/database.yml.mysql config/database.yml
$ chmod o-rwx config/database.yml
$ vi config/database.yml

#configure git defaults
$ git config --global user.name "GitLab"
$ git config --global user.email "dontreply@gitlab.example.com"
$ git config --global core.autocrlf input
#if you want to allow import from https sources with self-signed certificates
$ git config --global http.sslVerify false

#install all requiered dependencies
$ bundle install --deployment --without development test postgres aws

#install gitlab shell (handler for ssh connections)
$ bundle exec rake gitlab:shell:install[v2.2.0] REDIS_URL=unix:/var/lib/redis/redis.sock RAILS_ENV=production

#initialize mysql database
$ bundle exec rake gitlab:setup RAILS_ENV=production

#show the env information
$ bundle exec rake gitlab:env:info RAILS_ENV=production

#precompile the static assets
$ bundle exec rake assets:precompile RAILS_ENV=production

#get back to root context
$ exit
```

- Create init scripts, logrotate confs

```bash
#copy init script and config for it
$ cp /opt/gitlab/gitlab/lib/support/init.d/gitlab /etc/init.d/gitlab
$ cp /opt/gitlab/gitlab/lib/support/init.d/gitlab.default.example /etc/default/gitlab

#update configuration for init script
# app_root: "/opt/gitlab/gitlab"
$ sed -i 's/app_root=.*/app_root="\/opt\/gitlab\/gitlab"/' /etc/default/gitlab

#copy logrotate configuration and update the paths
$ cp lib/support/logrotate/gitlab /etc/logrotate.d/
$ sed -i 's/\/home\/git/\/opt\/gitlab/g' /etc/logrotate.d/gitlab
```

# Install nginx

- We need nginx >1.3.9, so lets use the offical package and not one from epel

```bash
#if you want take another version from http://nginx.org/packages/rhel/6/x86_64/RPMS/ or use configured yum repository to install it
$ wget -P /tmp http://nginx.org/packages/rhel/6/x86_64/RPMS/nginx-1.6.2-1.el6.ngx.x86_64.rpm
$ rpm -i /tmp/nginx-1.6.2-1.el6.ngx.x86_64.rpm

#configure pemissions, allow nginx to access gitlab
$ usermod -aG git nginx
$ chmod g+x /opt/gitlab

#copy the default nginx conf
$ cp /opt/gitlab/gitlab/lib/support/nginx/gitlab /etc/nginx/conf.d/default.conf
#change the paths
$ sed -i 's/\/home\/git/\/opt\/gitlab/g' /etc/nginx/conf.d/default.conf
#create SSL self-signed certs
$ mkdir /etc/nginx/ssl
$ openssl req -newkey rsa:4096 -x509 -nodes -days 3560 -out /etc/nginx/ssl/gitlab.crt -keyout /etc/nginx/ssl/gitlab.key
$ chmod o-r gitlab.key
```

# Configure nginx

- There are couple of issues with relative url setup (``/gitlab`` instead of ``/``), so as a workaround we serve some of static content directly via nginx
  - [All images in help section returns either "404 Not Found" or "500 Internal Server Error" response](https://github.com/gitlabhq/gitlabhq/issues/7610)
  - [No icons after 6.5 upgrade](https://github.com/gitlabhq/gitlabhq/issues/6216)
  - [Profile avatar photo seems broken after uploading file](https://github.com/gitlabhq/gitlabhq/issues/5430)
  - [Avatar not handled properly using relative path in 7.5.0.rc1](https://github.com/gitlabhq/gitlabhq/issues/8366)
  - [Images broken again after update...](https://github.com/sameersbn/docker-gitlab/issues/218)
- Check and update the nginx configuration so it looks like this

```text
# /etc/nginx/conf.d/default.conf
upstream gitlab {
  server unix:/opt/gitlab/gitlab/tmp/sockets/gitlab.socket fail_timeout=0;
}

## Normal HTTP host
server {
  listen *:443 default_server;
  server_name gitlab.example.com; ## Replace this with your server name
  server_tokens off; ## Don't show the nginx version number, a security best practice
  root /opt/gitlab/gitlab/public;

  ## Increase this if you want to upload large attachments
  ## Or if you want to accept large git objects over http
  client_max_body_size 512m;

  ## Individual nginx logs for this GitLab vhost
  access_log  /var/log/nginx/gitlab_access.log;
  error_log   /var/log/nginx/gitlab_error.log;

  location / {
    ## Serve static files from defined root folder.
    ## @gitlab is a named location for the upstream fallback, see below.
    try_files $uri $uri/index.html $uri.html @gitlab;
  }

  # workaround for images issues with relative urls, serve it via nginx
  # There are a lot of issue with relative urls in different places
  # They should be mostly solved with gitlab 7.6
  # Artem Sidorenko
  location ~ ^/gitlab/(assets|uploads)/ {
    rewrite /gitlab/(.*) /$1 break;
  }
  location ~ ^/gitlab/gitlab/ {
    rewrite /gitlab/gitlab/(.*) /$1 break;
  }
  location ~ ^/gitlab/help/.*\.png {
    root /opt/gitlab/gitlab/doc;
    rewrite /gitlab/help/(.*) /$1 break;
  }

  ## If a file, which is not found in the root folder is requested,
  ## then the proxy passes the request to the upsteam (gitlab unicorn).
  location @gitlab {
    ## If you use HTTPS make sure you disable gzip compression
    ## to be safe against BREACH attack.
    gzip off;

    ## https://github.com/gitlabhq/gitlabhq/issues/694
    ## Some requests take more than 30 seconds.
    proxy_read_timeout      600;
    proxy_connect_timeout   600;
    proxy_redirect          off;

    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto   $scheme;
    proxy_set_header    X-Frame-Options     SAMEORIGIN;

    proxy_pass http://gitlab;
  }

  ## Enable gzip compression as per rails guide:
  ## http://guides.rubyonrails.org/asset_pipeline.html#gzip-compression
  ## WARNING: If you are using relative urls remove the block below
  ## See config/application.rb under "Relative url support" for the list of
  ## other files that need to be changed for relative url support
#  location ~ ^/(assets)/ {
#    root /opt/gitlab/gitlab/public;
#    gzip_static on; # to serve pre-gzipped version
#    expires max;
#    add_header Cache-Control public;
#  }

  #if you have to block access to the graphs function create /var/www/html/error403.html and enable this part
  #location ~ ^/gitlab/[[:punct:][:alnum:]]*/[[:punct:][:alnum:]]*/graphs/[[:punct:][:alnum:]]*$ {
  #  root /var/www/html;
  #  deny all;
  #  error_page 403 /error403.html;
  #}
  #
  #location /error403.html {
  #  root /var/www/html;
  #  allow all;
  #}

  error_page 502 /502.html;

  #SSL
  ssl on;
  ssl_certificate /etc/nginx/ssl/gitlab.crt;
  ssl_certificate_key /etc/nginx/ssl/gitlab.key;
  ssl_session_timeout  5m;
  ssl_protocols  TLSv1;
  ssl_ciphers EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH+aRSA+RC4:EECDH:EDH+aRSA:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS:!RC4;
  ssl_prefer_server_ciphers   on;
}
```

# Enable and start everything

```bash
#enable and start gitlab
$ chkconfig --add gitlab
$ /etc/init.d/gitlab start
#enable and start nginx
$ chkconfig nginx on
$ /etc/init.d/nginx start
```

# Check the gitlab status

```bash
#first of all switch to the user git
$ sudo -u git -H -i bash
$ cd gitlab
#check the status for any failures
$ bundle exec rake gitlab:check RAILS_ENV=production
#get back to root context
$ exit
```
