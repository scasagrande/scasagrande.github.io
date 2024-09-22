+++
title = 'Compiling Arduino Caterina with New VID/PID'
slug = 'compiling-arduino-caterina-with-new-vidpid'
summary = 'How to compile a custom Arduino Caterina bootloader'
aliases = ['/articles/compiling-arduino-caterina-with-new-vidpid']
date = 2014-02-19
draft = false
+++

Compiling the Arduino Leonardo bootloader (aka Caterina) is a required step in making your own Arduino Leonardo-compatible board available to others. This blog post will go over what you need to do in order to make your derivative board valid for sale.

As opposed to previous Arduinos, the Leonardo uses the built-in USB of the atmega32u4 rather than a discrete USB-to-serial IC. All USB devices have two identification numbers: a vendor ID (VID) and a product ID (PID). Each organization that produces USB devices pays the USB-IF for a VID, giving them the ability to self-issue PIDs. Although the Arduino is open source hardware, this does not include the rights to use their VID in derivative products. You must obtain your own VID/PID pair and recompile the bootloader. In fact, you actually will need two PIDs for this project, one for the bootloader and one for the running sketch.

How you obtain your VID/PID pairs is beyond the scope of this article. I will offer one suggestion. If you release your project as open source, you can ask the folks at the Openmoko project to be issues a PID with their VID. You can see more details about that here: [http://wiki.openmoko.org/wiki/USB_Product_IDs](http://wiki.openmoko.org/wiki/USB_Product_IDs)

With your two VID/PID pairs in hand, lets compile the bootloader. I am going to be making the assumption that this is taking place on a Linux machine. The steps themselves will be very similar for compiling on a Windows machine, but require a few extra steps to get the path variables all setup. Honestly its a pain in the arse and I hate doing development work on Windows.

# Step 1

Make sure the Arduino IDE is installed. This ensures you have all the required tools as well as the source code.

# Step 2

Download LUFA-111009 and unpack it in the same directory as your caterina folder. By default on ubuntu, this will be `/usr/share/arduino/hardware/arduino/bootloaders`

# Step 3 (optional)

This step is optional. Open `Descriptors.c` located at `/usr/share/arduino/hardware/arduino/bootloaders/caterina/Descriptors.c` . Navigate down to line 192 (`const USB_Descriptor_String_t ProductString`). Here you can add a product ID string for your PID. For example, I added:

```c
#elif DEVICE_PID == 0x609C
.UnicodeString = L"Galvant antiAFk "
```

Directly below the Arduino Esplora entry. Notice line 194 which specifies a string length of 16 characters. If you entry is less, add some spaces on the end (as done with the Arduino Micro). If it is more, you will need to increase this value in one of two ways. Either increase it here at this line and add spaces to pad out all the existing Arduino strings, or move this line within the #if #endif block. Do the same for the VID entries directly below. Here is the entire block for the VID copied from my file:

```c
const USB_Descriptor_String_t ManufNameString =
{
    #if DEVICE_VID == 0x2341
    .Header = {.Size = USB_STRING_LEN(11), .Type = DTYPE_String},
    .UnicodeString = L"Arduino LLC"
    #elif DEVICE_VID == 0x1D50
    .Header = {.Size = USB_STRING_LEN(18), .Type = DTYPE_String},
    .UnicodeString = L"Galvant Industries"
    #else
    .Header = {.Size = USB_STRING_LEN(11), .Type = DTYPE_String},
    .UnicodeString = L"Unknown "
    #endif
};
```

# Step 4

Open the Makefile (`/usr/share/arduion/hardware/arduino/bootloaders/caterina/Makefile`) and look for the variable called “LUFA_PATH”. This should be around line 130. Modify this line to point to the folder LUFA-111009 that you extracted. If you have been following my instructions, this line should read `<code>LUFA_PATH = ../LUFA-111009</code>`

# Step 5

At the beginning of the Makefile after the file header is where the VID and PID are defined. Comment out any VID or PID currently specified and replace the entires with your VID and PID. The PID that you select here (out of your two) will be the one that is used for the bootloader. Mine looks like this:

```c
# USB vendor ID (VID)
# reuse of this VID by others is forbidden by USB-IF
# official Arduino LLC VID
# VID = 0x2341
# Openmoko VID
VID = 0x1D50

# USB product ID (PID)
# official Leonardo PID
# PID = 0x0036
# official Micro PID
# PID = 0x0037
# official Esplora PID
# PID = 0x003C
# antiAFK PID
PID = 0x60A3
```
    

# Step 6

Open your terminal and change directory to `/usr/share/arduino/hardware/arduino/bootloaders/caterina` and run make clean and make all. Depending on your account permissions this may have to be run as the super user. The compiled bootloader will be named `Caterina.hex`.

# Step 7

Rename `Caterina.hex` to something such as `Caterina-boardname.hex` so that it is not overridden in the future.

# Step 8

Open `boards.txt` (normally located at `/usr/share/arduino/hardware/arduino/boards.txt`). Copy and paste the entry for the Leonardo to the bottom of the file. Replace the leonardo at the beginning of each line with the name of your board and modify the rest of the entries as required. The key ones are the name (what you want it to show up as in the Arduino IDE) and `bootloader.file` (the name that you renamed `Caterina.hex` to in step 7).

# Step 9

Still at `boards.txt`, you want to change the `build.vid` and `build.pid` entries. For PID, you want to enter your OTHER PID, not the one you used in the earlier steps. This entry here is the VID/PID pair that will be used for the sketch which must be distinct from the bootloader PID. These changes to `boards.txt` must be located on any computer that will upload a sketch to your board in order to use to correct sketch VID/PID pair. Here is my boards.txt entry:

```ini
antiAFK.name=Galvant antiAFK
antiAFK.upload.protocol=avr109
antiAFK.upload.maximum_size=28672
antiAFK.upload.speed=57600
antiAFK.upload.disable_flushing=true
antiAFK.bootloader.low_fuses=0xff
antiAFK.bootloader.high_fuses=0xd8
antiAFK.bootloader.extended_fuses=0xcb
antiAFK.bootloader.path=caterina
antiAFK.bootloader.file=Caterina-antiAFK.hex
antiAFK.bootloader.unlock_bits=0x3F
antiAFK.bootloader.lock_bits=0x2F
antiAFK.build.mcu=atmega32u4
antiAFK.build.f_cpu=16000000L
antiAFK.build.vid=0x1D50
antiAFK.build.pid=0x609C
antiAFK.build.core=arduino
antiAFK.build.variant=leonardo
```
    

# Step 10

Burn the bootloader to your board! I just use my Arduino Uno and the Arduino IDE for this. To do the same, look up how to use the Uno as an ISP. Basically you upload the ArduinoISP example to your Uno, then select Tools->Programmer->Arduino as ISP, Tools->Serial Port->{your Uno}, Tools->Board->{which bootloader you want to burn}, then connect everything up and hit Tools->Burn Bootloader.

# Step 11

If you are running Linux, connect your new board to your PC and start uploading sketches! Sadly if you are running Windows, there is still a few more steps. You see, Windows does USB differently than the others. Linux looks at the enumeration and checks for standard types. If it is non-standard, it will then look at the VID/PID and search for a matching kernel module. Windows goes straight to looking at the VID/PID. So even though your device is functionally equivalent to the Arduino Leonardo, Windows will refuse to let you use it without a dedicated driver. An example driver file can be found [on GitHub](https://github.com/Galvant/usb_drivers/blob/master/Galvant_USB_driver.inf). Scroll down to the section “Vendor and Product ID Definitions” at line 78. Here, modify the beginning of each entry with the name of your product. IE, replace `antiAFK.bootloader` and `antiAFK.sketch` with `yourproduct.bootloader` and `yourproduct.sketch`. On each line there are two references to the VID and PID. Make you you replace all instances of the VID and PID with yours. There are two entries per line, and two lines per bootloader and sketch (one for 32 and one for 64 bit). Make super sure you have replaced ALL INSTANCES OF VID&PID WITH YOUR VALUES. Windows likes to cache USB information, even if you delete the driver, in the registry and this can be a pain in the arse to remove. MAKE SURE YOU PUT THE CORRECT VID/PID PAIR FOR BOOTLOADER AND THE CORRECT ONE FOR SKETCH. At the bottom on the file are a bunch of strings. Replace them as you see fit. Specifically, make sure to replace `MFGNAME` and `INSTDISK`, while also replacing `antiAFK.bootloader` and `antiAFK.sketch` with your own products and their strings.

# Step 12

On Windows, connect your device to your PC. Windows will complain about having no driver file. Go into the device manager (Start->Control Panel->Device Manager), right click on the device, update driver, and point it to the folder containing your INF file. Accept the dialog about the driver not being signed. Windows will then automatically use this file in the future for any USB devices with matching VID/PIDs pairs. You should now see in the device manager an entry for the bootloader. For my antiAFK, there would be an entry called “Galvant antiAFK Loader”. The first time you upload a sketch Windows will once again have to install the driver for the different sketch-PID. However, it will have your INF already stored and will do so automatically.

And you’re done! It’s a lot of steps, but I hope it helps.
