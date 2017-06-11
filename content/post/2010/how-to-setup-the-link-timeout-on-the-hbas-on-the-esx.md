+++
date = "2010-10-21T16:32:00+01:00"
tags = [ "vpshere", "esx", "san" ]
title = "How to setup the link timeout on the HBAs on the ESX"
+++

You may need it for example for an firmware upgrade of the disks on the storage system. Please ensure that you don't have huge IO in the time frame and you applications will not timeout because of missing disk and will retry disk IO operations.

<!--more-->

# Official documentation

From the [official VMWare documentation](http://www.vmware.com/pdf/vi3_san_design_deploy.pdf)

    Setting the HBA Timeout for Failover

    The timeout value for I/O retry operations is usually set in the HBA BIOS driver. (You
    might also want to change the operating system timeout, as discussed in “Setting
    Operating System Timeout” on page 148.)
    VMware recommends that you set the timeout value to 30 seconds:

       For QLogic HBAs, the timeout value is 2*n + 5 seconds, where n is the value of
       the PortDownRetryCount parameter of the BIOS of the QLogic card. You can
       change the path-failure detection time by changing the value of the module
       parameter qlport_down_retry (whose default value comes from the BIOS
       setting). The recommended setting for this parameter is 14.

       For Emulex HBAs, you can modify the path-failure detection time by changing the
       value of the module parameters lpfc_linkdown_tmo (the default is 30) and
       lpfc_nodedev_tmo (the default is 30). The sum of the values of these two
       parameters determines the path-failure detection time. The recommended setting
       for each is the default.

    To change these parameters, you must pass an extra option, such as
    qlport_down_retry or lpfc_linkdown_tmo, to the driver.

# Preparation of the ESX-Hosts

- Migrate all VM's to the other active nodes. Go into maintenance mode.
- Login via ssh and configure the timeout(QLogic HBAs):

```bash
esxcfg-module -s qlport_down_retry=300 qla2xxx.o
```

- you can show the actual options with

```bash
esxcfg-module -g qla2xxx.o
```

- Reboot the host and leave the maintenance mode.
- Do this with all ESX-Hosts in the environment.

After that you can start the firmware upgrade. In the tests, the system was running fine without a storage approximately 10 minutes.
