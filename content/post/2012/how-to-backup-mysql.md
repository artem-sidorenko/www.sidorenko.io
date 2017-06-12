+++
date = "2012-05-16T17:10:00+01:00"
tags = [ "mysql", "backup" ]
title = "How to backup mysql"
+++

We are going to create a simple cron job to dump the DBs, so you can save this dumps on the regular way with your backup software.

<!--more-->

**This way isn't applicable to DB installations with huge amount of data, so use it only for small DBs**

- Create a new user backup with global read(SELECT, LOCK TABLES) privileges

```bash
mysql -p
mysql> CREATE USER 'backup'@'localhost' IDENTIFIED BY  '<<some password>>';
mysql> GRANT SELECT,LOCK TABLES ON *.* TO 'backup'@'localhost';
mysql> FLUSH PRIVILEGES;
```

- Create a backup cronjob

```bash
mkdir /backup
echo "22 0 * * * root /usr/bin/mysqldump --all-databases -u backup --password=<<some password>> | gzip > /backup/mysql_dump.sql.gz" >> /etc/crontab
```
