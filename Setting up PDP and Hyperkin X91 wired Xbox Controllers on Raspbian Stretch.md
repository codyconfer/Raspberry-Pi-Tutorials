# Setting up PDP and Hyperkin X91 wired Xbox Controllers on Raspbian Stretch



[TOC]



## Overview

​	Raspbian uses a linux kernel module called xpad as the generic driver for Xbox One and Xbox 360 controllers. While the list of supported devices is extensive and expanding, sometimes the upstream changes in xpad don't ship with the current Raspbian image, or a device simply isn't registered in the kernel module. Registering a device in xpad is trivial, but can be daunting without direction. Some controllers can pick up out of the box support with xboxdrv, but in my case with a PDP wired Xbox one controller and a Hyperkin X91, I was not so lucky. This tutorial will walk through the steps of updating Raspbian, installing linux kernel headers, pulling the xpad source code from git, registering the vendorID and productID of the controllers in xpad.c and then reinstalling the kernel module with DKMS. It is a good idea to back up your system before this process.



## Updating Raspbian and Installing Dependencies

### Updating Raspbian

​	If you recently downloaded and installed Raspbian, you can skip this step. If you are plugging in a new controller to an existing machine, this step is essential as a kernel update may have added support for your device. Open up a terminal and run the following commands:

```bash
sudo apt-get update
sudo apt-get dist-upgrade
```

[raspberrypi.org](https://www.raspberrypi.org/documentation/raspbian/updating.md)



### Installing Kernel Headers

​	The kernel-headers are required to build kernel modules with DKMS. Kernel headers are included in the kernel's git repository, and they can be cloned from there. However, on Raspbian there is a package in the apt-get repository that results in easier set up. You can install kernel headers by running the following command:

```bash
sudo apt-get install raspberrypi-kernel-headers
```

[raspberrypi.org](https://www.raspberrypi.org/documentation/linux/kernel/headers.md)



### Install DKMS

​	DKMS (Dynamic Kernel Module System) is a tool to streamline building and installing linux kernel modules. This is the method this tutorial will use to build and install the modified xpad module. You can install DKMS by running the following command:

```bash
sudo apt-get install dkms
```

[DKMS GitHub](https://github.com/dell/dkms)



## Pulling and Modifying xpad.c Source Code

### Cloning Source

You can pull the xpad source from GitHub with the following command:

```bash
sudo git clone https://github.com/paroj/xpad.git /usr/src/xpad-0.4
```

You can now access the source by navigating to `/usr/src/xpad-0.4`:

```bash
cd /usr/src/xpad-0.4
```

[xpad GitHub](https://github.com/paroj/xpad)



### Getting VID and PID of Controllers

The vid and pid of your controller can be found through device manager in windows. 

​	1. Plug in the controller.

2. Right Click Start > Device Manager >  Xbox Peripherals > Rick Click Xbox Gaming Device > Properties > Events Tab
3. In the information field are the VID and PID for the device. `2E24 -> 0x2e24`.

In my case the ID's were as follows:

```json
PDP: {
    vid: 0x0e6f,
    pid: 0x02a7
}

X91: {
    vid= 0x2e24,
    pid= 0x1688
}
```

//img

### xpad.c Modifications

You can modify xpad on the pi using Geany, with su permissions.  In my case I had to check or modify 3 methods in the code base. The PDP controller required an extra step because of a payload the driver must deliver to the controller to complete intitialization. 

#### Check for or Register Device Vendor

Vendor registration occurs in `usb_device_id xpad_table[]`.

That struct looks something like this:

```c
static const struct usb_device_id xpad_table[] = {
    /* X-Box USB-IF not approved class */
	{ USB_INTERFACE_INFO('X', 'B', 0) },
    /* GPD Win 2 Controller */
	XPAD_XBOX360_VENDOR(0x0079),
    /* Thrustmaster X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x044f),
    /* Microsoft controllers */
	XPAD_XBOX360_VENDOR(0x045e),
	XPAD_XBOXONE_VENDOR(0x045e),
    /* Hyperkin Duke X-Box One pad */
	XPAD_XBOXONE_VENDOR(0x2e24),
    /* Logitech X-Box 360 style controllers */
	XPAD_XBOX360_VENDOR(0x046d),	
    /* Elecom JC-U3613M */
	XPAD_XBOX360_VENDOR(0x056e),
    /* Saitek P3600 */
	XPAD_XBOX360_VENDOR(0x06a3),
    /* Mad Catz X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x0738),	
    /* Mad Catz Beat Pad */
	{ USB_DEVICE(0x0738, 0x4540) },	
    /* Mad Catz FightStick TE 2 */
	XPAD_XBOXONE_VENDOR(0x0738),
    /* Mad Catz GamePad */
	XPAD_XBOX360_VENDOR(0x07ff),
    /* 0x0e6f X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x0e6f),	
    /* 0x0e6f X-Box One controllers */
	XPAD_XBOXONE_VENDOR(0x0e6f),	
    /* Hori Controllers */
	XPAD_XBOX360_VENDOR(0x0f0d),		
	XPAD_XBOXONE_VENDOR(0x0f0d),
    /* Nacon GC100XF */
	XPAD_XBOX360_VENDOR(0x11c9),	
    /* X-Box 360 dance pads */
	XPAD_XBOX360_VENDOR(0x12ab),
    /* RedOctane X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x1430),	
    /* BigBen Interactive Controllers */
	XPAD_XBOX360_VENDOR(0x146b),	
    /* Razer  */
	XPAD_XBOX360_VENDOR(0x1532),
	XPAD_XBOXONE_VENDOR(0x1532),
    /* Numark X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x15e4),
    /* Joytech X-Box 360 controllers */
	XPAD_XBOX360_VENDOR(0x162e),	
    /* Razer Onza */
	XPAD_XBOX360_VENDOR(0x1689),	
    /* Harminix Rock Band Guitar and Drums */
	XPAD_XBOX360_VENDOR(0x1bad),
    /* PowerA Controllers */
	XPAD_XBOX360_VENDOR(0x24c6),		
	XPAD_XBOXONE_VENDOR(0x24c6),
	{ }
};
```

In my case, both Hyperkin and PDP are already registered vendors, but the devices I have are not registered. Inadvertently adding a vendor twice will not negatively affect functionality.  You would register the vendor using the vid and XPAD_XBOX360_VENDOR() or XPAD_XBOXONE_VENDOR() depending on the console the controller targets.

#### Register Device

Product registrations occur in `xpad_device[]`

The struct contains an array of entries that look similar to:

```c
{ 0x2e24, 0x1688, "Hyperkin X91", 0, XTYPE_XBOXONE },
```

The schema is as follow: 

```c
static const struct xpad_device {
	u16 idVendor;
	u16 idProduct;
	char *name;
	u8 mapping;
	u8 xtype;
}
```

The name can be anything, the mapping has a default value of 0 and xtype is either :
`{ XTYPE_XBOX, XTYPE_XBOX360, XTYPE_XBOXONE }` depending on the on the controller's target console.

The two entries I added were :

```c
{ 0x2e24, 0x1688, "Hyperkin X91", 0, XTYPE_XBOXONE },
{ 0x0e6f, 0x02a7, "PDP Xbox One Controller", 0, XTYPE_XBOXONE },
```

#### Sending Config Payload (PDP)

On first attempt the Hyperkin controller worked, but the PDP did not. Upon further inspection of xpad.c `xboxone_init_packets[]` was found. 

`xboxone_init_packets[]` contained entries for other PDP product id's similar to:

```c
XBOXONE_INIT_PKT(0x0e6f, 0x02a8, xboxone_pdp_init1),
XBOXONE_INIT_PKT(0x0e6f, 0x02a8, xboxone_pdp_init2),
```

I adding the following lines for my controller's pid:

```c
XBOXONE_INIT_PKT(0x0e6f, 0x02a7, xboxone_pdp_init1),
XBOXONE_INIT_PKT(0x0e6f, 0x02a7, xboxone_pdp_init2),
```



## Updating xpad Kernel Module

If this is the first install, you can accomplish the build with the following command.

```bash
sudo dkms install -m xpad -v 0.4
```

If you are updating after initial installation, you should use the remove command before running install.
```bash
sudo dkms remove -m xpad -v 0.4 --all
sudo dkms install -m xpad -v 0.4
```


[xpad GitHub](https://github.com/paroj/xpad)



