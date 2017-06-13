+++
date = "2012-03-02T12:40:00+01:00"
tags = [ "mssql", "php" ]
title = "How to access MSSql from PHP via ODBC"
+++

# Install everything
- Install freetds for your distribution, you can find RH packages at [atrpms.net] (http://packages.atrpms.net/dist/el6/).
- Depending on your distribution you will need to install libtdsodbc
- Install php odbc extension

<!--more-->

# FreeTDS configuration

Create FreeTDS driver configuration in the file **freetds.driver.template**

```text
[FreeTDS]
Description = FreeTDS
Driver = /usr/lib64/libtdsodbc.so.0
```

Import this config

```bash
odbcinst -i -d -f freetds.driver.template
```

Create ODBC Data Source in the file **freetds.datasource.template**

```text
[MSSQLDB]
Driver = FreeTDS
Description = MSSQL Database
Trace = No
Server = <<server ip>>
Port = 1433
Database = <<database>>
```

Import it as system dcn

```bash
odbcinst -i -s -l -f freetds.datasource.template
```

Test the connection

```bash
isql -v MSSQLDB DBUSER DBPASS
```

# PHP sample usage

```php
<?php

# connect to a DSN "MSSQLDB" with a user "cheech" and password "chong"
$connect = odbc_connect("MSSQLDB", "cheech", "chong");

# query the users table for all fields
$query = "SELECT * FROM users";

# perform the query
$result = odbc_exec($connect, $query);

# print the data from the database
while(odbc_fetch_array($result))
    print_r($result);

# close the connection
odbc_close($connect);

?>
```

# See too

- [unixODBC - MS SQL Server/PHP](http://www.unixodbc.org/doc/FreeTDS.html)
