# Reference Application Introduction:

This Reference Application is targetted specifically at the RD1552 Reference Design and is built on mbedOS and uses the "mbed-cloud-client" library to handle the Pelion Platform connection, registration and LWM2M resources. The application simulates a button press every 5 seconds and relays the packet to Arm Pelion IoT platform.

The current version of the Reference Application uses: Mbed OS v5.9.6 and Mbed Cloud Client v1.3.3.

Note: references to Mbed Cloud and Pelion Device Managament are interchangeable.  For example: “Mbed Cloud Client enables the device to communicate with Pelion Device Management” and “Mbed Cloud Client enables the device to communicate with Mbed Cloud” have the same meaning.  In later versions we will remove references to Mbed Cloud inline with branding guidelines.

# Pre-requisites:
1. Hardware:
    * RD1168 based board or platform (the application was developed and tested using the Example Implemenation Hardware - MCB/MTB
    * A micro USB cable
    * A WiFi access point

2. Software:
    * mbed CLI - https://github.com/ARMmbed/mbed-cli
    * manifest tool - ``` pip install git+https://github.com/ARMmbed/manifest-tool --upgrade ```
    * Both the above tools will be combined into mbed CLI (via command ``` dm ``` in a later release of mbed CLI)
    * Pelion account for API keys & developer certificate.

# Getting started:
1. Import this example to your local machine with mbed CLI:
    ``` mbed import https://github.com/ARMmbed/reference-designs.git ```

2. Move into the imported directory ``` cd <path to directory>```
    * Login to your Pelion Cloud account on a browser & follow the steps below:
        * Navigate to Device identity > Certificates
        * Select the certificate created by your account admin and click on "Download Developer C file"
        * Save the file?mbed_cloud_dev_credentials.c to the location of the RD1552 directory. You have to overwrite the contents of the existing file there.

3. Configure your network credentials:
    * In the mbed_app.json file in the root of the application, modify the WiFi SSID and password for your network.
    * Save the changes.

4. The supplied Bootloader in ../tools/ is currently built to use SPI Flash. If you are using the SPI Flash to store the certificates, then you are good to go. If you prefer to use the SD card for the storage, follow the steps below. If not, you may skip these from step 5 and continue from step 12.
   
5. To build the bootloader to use the SD card: TIP: You may want to use atleast a Class 10, 2GB SD card.
    * Import the bootloader repo - https://github.com/ARMmbed/mbed-bootloader
    * Change main.cpp with:
    ```
#if MBED_CLOUD_CLIENT_UPDATE_STORAGE == ARM_UCP_FLASHIAP_BLOCKDEVICE
#include "SDBlockDevice.h"  
/* initialise sd card blockdevice */  
#if defined(MBED_CONF_APP_SPI_MOSI) && defined(MBED_CONF_APP_SPI_MISO) && \  
defined(MBED_CONF_APP_SPI_CLK)  && defined(MBED_CONF_APP_SPI_CS)  
SDBlockDevice sd(MBED_CONF_APP_SPI_MOSI, MBED_CONF_APP_SPI_MISO,  
MBED_CONF_APP_SPI_CLK,  MBED_CONF_APP_SPI_CS);  
```

    * Modify mbed_app.json in bootloader: Change update storage-address and remove SPI pin defines.  
    
    ```
    "update-client.storage-address"  : "(1024*1024*64)", 
	```
	And

	```
        "MTB_UBLOX_ODIN_W2": {
		    "target.device_has_remove": ["EMAC"],
            "flash-start-address"              : "0x08000000",
            "flash-size"                       : "(2048*1024)",
            "sotp-section-1-address"           : "(MBED_CONF_APP_FLASH_START_ADDRESS+1792*1024)",
            "sotp-section-1-size"              : "(128*1024)",
            "sotp-section-2-address"           : "(MBED_CONF_APP_FLASH_START_ADDRESS+1920*1024)",
            "sotp-section-2-size"              : "(128*1024)",
            "update-client.application-details": "(MBED_CONF_APP_FLASH_START_ADDRESS+64*1024)",
            "application-start-address"        : "(MBED_CONF_APP_FLASH_START_ADDRESS+65*1024)",
            "max-application-size"             : "DEFAULT_MAX_APPLICATION_SIZE",
        },
   ```
6. Change mbed_app.json in source:
 ```   "update-client.storage-address"  : "(1024*1024*64)",  ```
   
7. Once changes are made in bootloader, build it:
``` mbed compile -t GCC_ARM -m MTB_UBLOX_ODIN_W2 ```

8. Copy the generated binary to ../tools/ directory. Rename to match the existing file name.

9. Change mbed_app.json in the root of the reference application i.e. 
"update-client.storage-address"  : "(1024*1024*64)"

10. Change main.cpp in source to match SD card
### IMPORTANT: The required lines are already supplied in the code. Uncomment appropriate lines in main.cpp and build the binary.

11. Combine with the bootloader using the supplied script in ../tools/ .
``` $> python tools/combine_bootloader_with_app.py -m MTB_UBLOX_ODIN_W2  -a BUILD/MTB_UBLOX_ODIN_W2/GCC_ARM/mbed-cloud-example.bin -o combined.bin ```

12. Flash the combined binary (combined.bin) to your device. Open a terminal & observe the output.

13. The device starts the example, connects to the WiFi network & after a few moments connects & registers to Pelion Cloud. It then prints out a device ID. You will need this for firmware update at a later stage.

* You can now check your simulated button counter's value in the Pelion portal as well.
* Navigate to "Device Directory" in the portal and click on your device's device ID. This will open up a new drawer on the page.
* Scroll down to LWM2M resource "Button_count" and click on it.
* A new window opens and you will be able to see the counter & a graph increment every time the resource value increments.

14. Congratulations, your device is now connected to Pelion IoT platform. Please proceed to the FW update section now.

## Recommendation:  
Increase stack size:         "MBED_CONF_APP_MAIN_STACK_SIZE=5000", to 6K if using newer versions of Mbed OS  


# Demonstrate a remote Firmware update:

1. In order to update FW on a connected device, you will need the manifest tool. Please note that in a later release of Mbed OS (5.10 & above) & Mbed CLI (1.8.0 and above), the manifest tool is integrated into the Mbed CLI (via the dm command) and you wouldn't need to manually install this. But, if using earlier versions then, you have to manually install this using:

 ``` pip install git+https://github.com/ARMmbed/manifest-tool --upgrade ```

2. A dependency is to install the python SDK along with the manifest tool. This is done using : ``` pip install mbed-cloud-sdk --upgrade ```

3. Now you need to setup Mbed OS on the device to receive FW updates:
	* Login to your Pelion Cloud portal.
	* Create an API device key (Access Management -> API Keys -> Create a new API key)
	* Ensure you copy the API key generated as this is visible only once.
	
4. Initialize the certificates for your application:
## IMPORTANT: You must initialize the certificates before you flash the device for the 1st time as these certificates need to be embedded within the device in order for it to receive remote updates.

``` manifest-tool init -a <api key> -S <mbed cloud API URL> -d <domain name> -m <model ID> --force ``` 

Where mbed cloud API URL is https://api.us-east-1.mbedcloud.com/  . The domain name must have a ".com" as a mandatory requirement and the model ID can be an integer of your choice.

Note the use of --force as this option overrides the defaults provided in the application. This is a mandatory parameter.

5. With the certificates initialized, there will now be a ".certificates" directory created within the application's root directory.

6. Now, compile the application:
``` mbed compile -t GCC_ARM -m MTB_UBLOX_ODIN_W2 ```

7. Combine the application with the bootloader - use the provided one in the \tools\ directory if you are using the SPI flash. If you have built a new bootloader to use the SD card, then you need to use the new bootloader.

``` python tools/combine_bootloader_with_app.py -m MTB_UBLOX_ODIN_W2 -a BUILD/MTB_UBLOX_ODIN_W2/GCC_ARM/reference-design-RD1552.bin -o combined.bin ```

8. Program your device with the ``` combined.bin ``` just generated. You can either drag-drop via the GUI of your OS or you can use the corresponding copy command from the terminal for your OS.

9. Open a serial terminal application such as PuTTY and observe the output. You will notice the device boots up, connects to the WiFi network, starts Pelion Cloud Client and registers to the Cloud.

10. Once the device registers, make a note of the device ID assigned to the device.

11. Modify the main.cpp in the source directory to add a small printf statement - this is already provided in the code. So, you can un-comment this line in order to demo a FW update.

12. Compile the new source with ``` mbed compile -t GCC_ARM -m MTB_UBLOX_ODIN_W2 ``` command.

13. You will now have a new binary in the BUILD\MTB_UBLOX_ODIN_W2\GCC_ARM\ directory.

14. Use the manifest-tool to perform a remote update on the device:

``` manifest-tool update device -p <payload name> -D <device id> ``` 

where payload name is the full path to your new binary built from step 13 and device ID is the one obtained from step 9.

15. You will notice from the serial terminal that the device gets a request to update the FW, then this is authorized, then the download of the new FW starts, finishes and the bootloader verifies the authencity of the new FW, installs it on the device and reboots.

16. You will now notice the new printf statement appear on the device logs which indicates that the newly built application binary is now installed and running on the device i.e. the device has been remotely updated.

The success logs should look like this on a serial terminal, although your device *will* be assigned a different device ID and will have a different IP address.

```
tegrity check:
[ERR ] Active firmware inte¦
BOOT] Active firmware integrity check:
[BOOT] [++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++]
[BOOT] SHA256: BA2A6CC9182829733B56D41DD068021EBBFCC5602EEFB287BFA941079A8E47B7
[BOOT] Version: 1537273612
[BOOT] Slot 0 is empty
[BOOT] Active firmware up-to-date
[BOOT] Application's start address: 0x8010400
[BOOT] Application's jump address: 0x80110B1
[BOOT] Application's stack address: 0x20030000
[BOOT] Forwarding to application...
¦¦¦*********** erase_size = 1024
Starting Simple Mbed Cloud Client example
Connecting to the network using WiFi...
Init
Connected to the network successfully. IP address: 192.168.0.101
[Simple Cloud Client] Autoformatting the storage.
*********** erase_size = 1024
*********** erase_size = 1024
[Simple Cloud Client] Reset storage to an empty state.
[Simple Cloud Client] Starting developer flow
client_status = 0
Initialized Mbed Cloud Client. Registering...
Simulated button clicked 1 times
Simulated button clicked 2 times
Connected to Mbed Cloud. Endpoint Name: 0165eca72f1100000000000100100016
Simulated button clicked 3 times
Simulated button clicked 4 times
Simulated button clicked 5 times
Simulated button clicked 6 times
Simulated button clicked 7 times
Simulated button clicked 8 times
Simulated button clicked 9 times
Simulated button clicked 10 times
Firmware download requested
Authorization granted
Simulated button clicked 11 times
Simulated button clicked 12 times
Downloading: [/                                                 ] 0 %Simulated button clicked 13 times
Downloading: [+/                                                ] 2 %Simulated button clicked 14 times
Downloading: [++-                                               ] 4 %Simulated button clicked 15 times
Downloading: [++-                                               ] 5 %Simulated button clicked 16 times
Downloading: [+++-                                              ] 7 %Simulated button clicked 17 times
Downloading: [++++|                                             ] 9 %Simulated button clicked 18 times
Downloading: [+++++\                                            ] 11 %Simulated button clicked 19 times
Downloading: [++++++|                                           ] 13 %Simulated button clicked 20 times
Downloading: [+++++++|                                          ] 15 %Simulated button clicked 21 times
Downloading: [++++++++-                                         ] 17 %Simulated button clicked 22 times
Downloading: [+++++++++-                                        ] 19 %Simulated button clicked 23 times
Downloading: [++++++++++|                                       ] 21 %Simulated button clicked 24 times
Downloading: [+++++++++++-                                      ] 23 %Simulated button clicked 25 times
Downloading: [++++++++++++/                                     ] 25 %Simulated button clicked 26 times
Downloading: [+++++++++++++-                                    ] 27 %Simulated button clicked 27 times
Downloading: [++++++++++++++\                                   ] 29 %Simulated button clicked 28 times
Downloading: [++++++++++++++++-                                 ] 32 %Simulated button clicked 29 times
Downloading: [+++++++++++++++++/                                ] 34 %Simulated button clicked 30 times
Downloading: [+++++++++++++++++\                                ] 35 %Simulated button clicked 31 times
Downloading: [++++++++++++++++++\                               ] 37 %Simulated button clicked 32 times
Downloading: [+++++++++++++++++++-                              ] 39 %Simulated button clicked 33 times
Downloading: [++++++++++++++++++++/                             ] 41 %Simulated button clicked 34 times
Downloading: [+++++++++++++++++++++\                            ] 43 %Simulated button clicked 35 times
Downloading: [++++++++++++++++++++++\                           ] 45 %Simulated button clicked 36 times
Downloading: [+++++++++++++++++++++++|                          ] 47 %Simulated button clicked 37 times
Downloading: [++++++++++++++++++++++++\                         ] 49 %Simulated button clicked 38 times
Downloading: [++++++++++++++++++++++++++/                       ] 52 %Simulated button clicked 39 times
Downloading: [+++++++++++++++++++++++++++-                      ] 54 %Simulated button clicked 40 times
Downloading: [+++++++++++++++++++++++++++-                      ] 55 %Simulated button clicked 41 times
Downloading: [++++++++++++++++++++++++++++\                     ] 57 %Simulated button clicked 42 times
Downloading: [+++++++++++++++++++++++++++++-                    ] 59 %Simulated button clicked 43 times
Downloading: [++++++++++++++++++++++++++++++-                   ] 61 %Simulated button clicked 44 times
Downloading: [+++++++++++++++++++++++++++++++-                  ] 63 %Simulated button clicked 45 times
Downloading: [++++++++++++++++++++++++++++++++|                 ] 65 %Simulated button clicked 46 times
Downloading: [+++++++++++++++++++++++++++++++++\                ] 67 %Simulated button clicked 47 times
Downloading: [++++++++++++++++++++++++++++++++++/               ] 69 %Simulated button clicked 48 times
Downloading: [+++++++++++++++++++++++++++++++++++|              ] 71 %Simulated button clicked 49 times
Downloading: [++++++++++++++++++++++++++++++++++++|             ] 73 %Simulated button clicked 50 times
Downloading: [+++++++++++++++++++++++++++++++++++++|            ] 75 %Simulated button clicked 51 times
Downloading: [++++++++++++++++++++++++++++++++++++++\           ] 77 %Simulated button clicked 52 times
Downloading: [+++++++++++++++++++++++++++++++++++++++/          ] 79 %Simulated button clicked 53 times
Downloading: [++++++++++++++++++++++++++++++++++++++++\         ] 81 %Simulated button clicked 54 times
Downloading: [+++++++++++++++++++++++++++++++++++++++++/        ] 83 %Simulated button clicked 55 times
Downloading: [++++++++++++++++++++++++++++++++++++++++++-       ] 85 %Simulated button clicked 56 times
Downloading: [+++++++++++++++++++++++++++++++++++++++++++-      ] 87 %Simulated button clicked 57 times
Downloading: [++++++++++++++++++++++++++++++++++++++++++++|     ] 89 %Simulated button clicked 58 times
Downloading: [+++++++++++++++++++++++++++++++++++++++++++++\    ] 91 %Simulated button clicked 59 times
Downloading: [++++++++++++++++++++++++++++++++++++++++++++++-   ] 93 %Simulated button clicked 60 times
Downloading: [+++++++++++++++++++++++++++++++++++++++++++++++/  ] 95 %Simulated button clicked 61 times
Downloading: [++++++++++++++++++++++++++++++++++++++++++++++++\ ] 97 %Simulated button clicked 62 times
Downloading: [+++++++++++++++++++++++++++++++++++++++++++++++++-] 99 %Simulated button clicked 63 times
Downloading: [++++++++++++++++++++++++++++++++++++++++++++++++++] 100 %
Download completed
Firmware install requested
[BOOT] mbed Bootloader
[BOOT] ARM: 0000000000000000000000000000000000000000
[BOOT] OEM: 0000000000000000000000000000000000000000
[BOOT] Layout: 0 800A85C
[BOOT] Active firmware integrity check:
[BOOT] [++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++]
[BOOT] SHA256: BA2A6CC9182829733B56D41DD068021EBBFCC5602EEFB287BFA941079A8E47B7
[BOOT] Version: 1537273612
[BOOT] Slot 0 firmware integrity check:
[BOOT] [++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++]
[BOOT] SHA256: 47D2FFB7DBAD19AFD1324F5D64DC891C35A75F18CEB4025FB781E50782F7AC2A
[BOOT] Version: 1537273748
[BOOT] Update active firmware using slot 0:
[BOOT] [++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++]
[BOOT] Verify new active firmware:
[BOOT] [++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++]
[BOOT] New active firmware is valid
[BOOT] Application's start address: 0x8010400
[BOOT] Application's jump address: 0x80110B1
[BOOT] Application's stack address: 0x20030000
[BOOT] Forwarding to application...
¦¦¦*********** erase_size = 1024
Starting Simple Mbed Cloud Client example
Connecting to the network using WiFi...
Init
Connected to the network successfully. IP address: 192.168.0.101

*********This line demonstrates a firmware update*********
[Simple Cloud Client] Starting developer flow
[Simple Cloud Client] Developer credentials already exist
client_status = 0
Initialized Mbed Cloud Client. Registering...
Connected to Mbed Cloud. Endpoint Name: 0165eca72f1100000000000100100016
Simulated button clicked 1 times
Simulated button clicked 2 times
Simulated button clicked 3 times

```

### Also note that the device ID remains the same after the FW update. This indicates that the SOTP regions were not over-written while perfoming the update.