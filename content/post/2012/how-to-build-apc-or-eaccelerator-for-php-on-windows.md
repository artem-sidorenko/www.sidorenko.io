+++
date = "2012-04-20T21:49:47+01:00"
tags = [ "php" ]
title = "How to build APC or eAccelerator for PHP on windows"
+++


If you running some php web applications with heavy load you should use some accelerators like [APC](http://pecl.php.net/package/APC) or [eAccelerator](https://eaccelerator.net/) for PHP.
I noticed that official build instructions in readme's and official homepages are not completely usable to get it working.

<!--more-->

So, we are going to build shared APC extension(php_apc.dll) or eAccelerator extension(eAccelerator.dll) for PHP on windows. Our build platform will be Windows XP SP3 32bit and we are using PHP 5.3.10 VC9 thread safe 32bit.

I had a bad experience with APC on Windows. I don't know why, but the web server was unstable and didn't proceed any queries after some time. eAccelerator is working without any problems. Anyway, lets go

# Configure PHP build enviroment

- [Download](http://www.microsoft.com/visualstudio/en-us/products/2008-editions/express) and install Visual C++ 2008 Express Edition
- [Download](http://www.microsoft.com/download/en/details.aspx?displaylang=en&id=11310#Overview) and install Windows SDK 6.1
- Create C:\php-sdk\
- [Download](http://windows.php.net/downloads/php-sdk/) and unpack sdk-binary-tools to C:\php-sdk\
- open the “windows sdk 6.1 shell” (available in Start menu) and type following commands, don't close the shell

```bash
setenv /x86 /xp /release
cd c:\php-sdk\
bin\phpsdk_setvars.bat
bin\phpsdk_buildtree.bat php53dev
```

- [Download](http://php.net/downloads.php) php 5.3.10 sources and unpack it to C:\php-sdk\php53dev\vc9\x86, so you get a new folder C:\php-sdk\php53dev\vc9\x86\php5.3.10
- [Download](http://windows.php.net/downloads/php-sdk/deps-5.3-vc9-x86.7z) php libs and unpack it to C:\php-sdk\php53dev\vc9\x86\deps

# Build APC

- [Download](http://pecl.php.net/package/APC) the last APC version and unpack the content of APC-xyz it to C:\php-sdk\php53dev\vc9\x86\php5.3.10\ext\apc
- Run following commands in the windows sdk shell

```bash
cd C:\php-sdk\php53dev\vc9\x86\php5.3.10
buildconf
configure --enable-apc=shared
nmake php_apc.dll
```

- You can find your php_apc.dll in C:\php-sdk\php53dev\vc9\x86\php5.3.10\Release_TS

# Build eAccelerator

- [Download](http://sourceforge.net/projects/eaccelerator/) the last eAccelerator version and unpack the content of eaccelerator-xyz to C:\php-sdk\php53dev\vc9\x86\php5.3.10\ext\eaccelerator
- [Download](http://windows.php.net/download/) php 5.3.10 VC9 thread safe binaries, copy dev/php5ts.lib to C:\php-sdk\php53dev\vc9\x86\php5.3.10
- Run following commands in the windows sdk shell

```bash
cd C:\php-sdk\php53dev\vc9\x86\php5.3.10
buildconf
configure
```

- Go to C:\php-sdk\php53dev\vc9\x86\php-5.3.10\ext\eaccelerator\win32 and open eAccelerator.dsw with Visual Studio
- Go to Build - Configuration manager and switch the active solution configuration to release
- Build the DLL: Build - Build eAccelerator
- You can find your eAccelerator.dll in C:\php-sdk\php53dev\vc9\x86\php-5.3.10\ext\eaccelerator\win32\Release

# See too

- [Adding a PECL extension to your PHP build environment](http://www.ksingla.net/2010/05/adding-a-pecl-extension-to-your-php-build-environment/)
- [Build your own PHP on Windows](https://wiki.php.net/internals/windows/stepbystepbuild)
