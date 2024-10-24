# Linux-on-MBP-14-2
My process for getting Debian 12 working on a Macbook Pro 14,2.  Much of this information came from [Dunedan's](https://github.com/Dunedan/mbp-2016-linux) wonderful guide on the state of linux on MBPs, though some of this information I had to look for elsewhere or try for myself.  Though it's somewhat old and outdated, the guide was a good way to try and find solutions to the problems I ran into along the way

## Needed Fixing
Audio - Internal speakers, microphone, and 3.5mm jack not working
Bluetooth - Mostly works on startup, sometimes doesn't and requires a system reboot
WiFi - Poor signal, can't connect to networks
Touchbar - Not turning on at all
Suspension/Hibernation - Computer would never wake from suspension

## Audio
Used [davidjo's](https://github.com/davidjo/snd_hda_macbookpro) snd_hda audio drivers for MBP's with a CS8409 audio chip (includes MBP 14,2s)
Was able to run the installer as directed in their README, and after rebooting, I needed to head to Audio and set the Speakers profile to "Analoge Stereo Duplex"
After that, the internal speakers, microphone, and audio jack would work.

## WiFi
Followed the guide from [nonylus](https://bugzilla.kernel.org/show_bug.cgi?id=193121#c62), downloading [brcmfmac43602-pcie.txt](https://bugzilla.kernel.org/attachment.cgi?id=285753), changed the values of "macaddr" to "macaddr=00:90:4c:0d:f4:3e", "ccode" to "ccode=0", and "regrev" to "regrev=1".  I then put this modified file alongside the wifi drivers at /lib/firmware/bcrm/, then rebooted

## Touchbar
Used a personally modified version of [almas'](https://github.com/almas/macbook12-spi-driver/tree/touchbar-driver-hid-driver), (a branch of [roadrunner2's(https://github.com/roadrunner2/macbook12-spi-driver)] touchbar drivers), and modified some of the files as almas' version wouldn't compile.
In order for them to compile for me, I modified the file "apple-ibridge.c"

--- a/apple-ib-tb.c
+++ b/apple-ib-tb.c
@@ -951,8 +951,10
	switch (field->report->type) {
	case HID_INPUT_REPORT:
-		report_info->report_type = 0x01; break;
+		report_info->report_type = 1; break;
	case HID_OUTPUT_REPORT:
-		report_info->report_type = 0x02; break;
+		report_info->report_type = 2; break;
	case HID_FEATURE_REPORT:
-		report_info->report_type = 0x03; break;
+		report_info->report_type = 3; break;
+	default:
+		break;
	}
--- a/apple-ibridge.c
+++ b/apple-ibridge.c
@@ -846,8 +846,8
-	static void appleib_remove(struct acpi_device *acpi)
+	static int appleib_remove(struct acpi_device *acpi)
	{
		struct appleib_device *ib_dev = acpi_driver_data(acpi);

		hid_unregister_driver(&ib_dev->ib_driver);

-		return;
+		return 1;
	}

The original touchbar drivers would fail to compile as line 905 of apple-ibridge.c would attempt to assign a value '.remove' with the return address from appleib_remove, which is a void function and thusly, doesn't return anything.  The change now has it return an integer '1'.  The file apple-ib-tb.c would crash as the program expected a default case in the switch case and expected the variables report_type to be assigned integers.  Simply changing the values '0x0n' to 'n' would fix the problem.
After modifying those two lines, the drivers compiled for me without problem, though still had problems staring.  I can not find the exact source I saw this right now but a comment on a issue under one of the forked versions of the touchbar drivers mentioned that usbmuxd would have trouble interpretting the touchbar on the ibridge and required for the usb to be unbound and rebounded before the touchbar would work.  This is done by running `sudo echo '1-3' > /sys/bus/usb/drivers/usb/unbind && sudo echo '1-3' > /sys/bus/usb/drivers/usb/bind` anytime, though presumably on or shortly after startup to active the touchbar.





## Suspension/Hibernation
The NVMe controller that handles the internal storage drive would not successfully wake up as it needs the d3cold PCIe power state to do so.  This is corrected by running `sudo echo 0 > /sys/bus/pci/devices/0000:01:00.0/d3cold_allowed` on or shortly after startup.  I still would not recommend suspending the computer as it seems to cause problems on wake, specifically I've had a harder time connecting to networks and bluetooth, though I'm sure there are other problems cause by this that I haven't noticed.  I'd recommend avoiding suspending your computer, though it's still good to run the command in case of emergency
