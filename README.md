# RuuviCollector Rasbian

This project contains the setup for a Rasbian Stretch Lite based image that has been set up to run the RuuviCollector java jar in order to collect data from various RuuviTags. 

## Pre-built images

The image contains dependencies that are processor architecture dependent, so one should use the pre-built image that is suitable for the architecture.

- Rasbian Strech Lite on arm32v6 (Raspberry Pi v.1, Rasperry Pi Zero)


## Setup using the pre-built release

1. Download a release image from this project and flash it to an SD card the same way you'd flash the official Rasbian image. I recommende Etcher for Mac OS.

1. Add a file called ssh to the root of the ```boot``` partition.
1. Setup Wifi if your board supports it or you use a wifi dongle.
    ```
    sudo raspi-config
    ```

    For headless setup, create a file called ```wpa_supplicant.conf``` with the content below and place it in the ```boot``` partition (same as when acticating ssh)

    ```
    country=FI
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1

    network={
        ssid="XXXX"
        scan_ssid=1
        psk="your_real_password"
        key_mgmt=WPA-PSK    
    }
    ```

1. Create the ruuvi-collector.properties and ruuvi-names.properties files and place them in the ```boot``` partition (same as the ssh file). These files will be copied to the correct location at startup, so whenever you need to do a change, simply modify the properties files here and reboot. See the  [RuuviCollector](https://github.com/Scrin/RuuviCollector/) project for the content of these files

1. Provided the properties files and wifi config is correct and you have a influxdb reachable by the RPi, this should be enough to start the system collecting data. 

Note: there may be an issue in which the .properties files are removed at first boot and thus not copied to the correct place. Simply create the files again and re-boot should fix it until I figure out what happens.

1. Once the Raspberry Pi is up and running, first step should be to ssh into the device and changing the default password (the default is the same as for the official image)! While you're at it, why not add your own SSH key as well. The machine should be reachable at the address ```collectorpi.local```

## Troubleshooting

If using a Bluetooth dongle, which USB port one connects the dongle in may matter, at least on a RPi v1. So if ```hcitool lescan``` and/or ```hcidump --raw``` do not produce any output, make sure the device is listed when running ```lsusb```. If not, change ports and try again.

To check status of collector service run: ```systemctl status ruuvi-collector``` (The service starts only after everything else has started, so may take a while before it shows status active running)

Sometimes it helps to restart the bluetooth daemon by running
```
sudo service bluetooth restart
```

Another thing to try is
```
hciconfig hci0 down
hciconfig hci0 up
```

## Image build steps

### Install of packages on running system
1. Download latest Raspbian Stretch Lite and flash the image to SD card using e.g. Etcher.
1. Mount the image back to the computer and enable ssh by creating a file called ```ssh``` in the root of the boot partition
1. Start up and SSH into the device using default credentials (find IP using command e.g. ```arp -a``` on Mac OS)
1. Install some dependencies and tools (vim, Java jre): 
    ```
    sudo apt-get update && \
    sudo apt-get install -y vim openjdk-8-jre-headless openjdk-8-jre bluez bluez-hcidump && \
    sudo apt-get clean
    ```
1. Hostname changed to collectorpi
    ```
    sudo vi /etc/hostname
    sudo vi /etc/hosts
    ```

    In both files, change raspberrypi to collectorpi

1. Setup the RuuviCollector dependencies (on the raspberry pi)
    ```
    sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcitool`
    sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcidump`
    mkdir -p /home/pi/collector
    ```

1. Copy ruuvi-collector-*.jar and service deps to raspbian (version 0.2) 
   ```
   rsync -avz lib/* pi@raspberrypi.local:/home/pi/collector/.
   ```

1. Setup service scripts
   ```
   rsync -avz service/ruuvi-collector.service pi@raspberrypi.local:/home/pi/collector/.
   rsync -avz service/ruuvi-collector.service pi@raspberrypi.local:/home/pi/collector/.
   sudo cp /home/pi/collector/ruuvi-*.service /etc/systemd/system/.
   sudo systemctl enable ruuvi-collector
   sudo systemctl start ruuvi-collector
   sudo systemctl enable ruuvi-config-copy
   sudo systemctl start ruuvi-config-copy
   ```

   The last step is just to test that the service can be run at some level. Without the properties files in place it will fail, but those should not be part of the image build as they contain sensitive data.

1. Clear history and logout
    ```

    ```

### Create distributable image

You need a Linux machine or VM for this. Steps below were done on a Ubuntu 18.04 machine.

1. Install gparted if not already installed
   ```
   sudo apt-get install gparted
   ```

1. Install pishrink if not already installed
   ```
   wget  https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
   chmod +x pishrink.sh
   sudo mv pishrink.sh /usr/local/bin
   ```

1. Insert the sd-card with the raspbian to a sd-card reader on the Ubuntu machine, then

1. Clone the sdhc card to an image
    ```
    sudo fdisk -l
    sudo dd if=/dev/<path to sd card as seen in fdisk output> of=/your/path/to/clone.img status=progress
    ```

    This will take a while depending on the sd card size.

1. Shrink the image
   ```
   sudo pishrink.sh /path/to/clone.img /path/to/clone-shrinked.img
   ```

   Again, this will take a while. Once done, to pack the img even further, zip it
   ```
   zip /path/to/clone-shrinked.img -O /path/to/clone-shrinked.zip
   ```

1. Distribute to the world


