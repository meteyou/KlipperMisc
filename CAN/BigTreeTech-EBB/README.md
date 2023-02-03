# BigTreeTech EBB

<br>

<p>This is an instruction to set up the BTT EBB36, EBB42 and EBB S<i>(tealth)</i>B<i>(urner)</i> with Klipper via CANBUS.</p>
<br>
<p>This guide was verified on a Pi running <a href="https://github.com/mainsail-crew/MainsailOS">MainsailOS</a></p>
<br>

> ⚠ Use at least a Klipper version of v0.10.0-531 to use the board safely! \
> In [this commit](https://github.com/Klipper3d/klipper/commit/3796a319599e84b58886ec6f733277bfe4f1a747), Kevin fixed a bug in the ADC calculation of the STM32G0.

<br>
<br>
<br>

<h2>
  <b>Instruction Index:</b>
</h2>

<br>

<ul>
  <li><a href="#canboot">CanBoot bootloader <i>(optional)</i></a></li>
  <ul>
    <li><a href="#canboot-download">Download</a></li>
    <li><a href="#canboot-config">Configuration</a></li>
    <li><a href="#canboot-flash">Flashing</a></li>
    <br>
    <li><a href="#canboot-update">Updating</a></li>
  </ul>
  <br>
  <br>
  <li><a href=" ">Klipper</a></li>
  <ul>
    <li><a href="#klipper-config">Configuration</a></li>
    <li><a href="#klipper-flash">Flashing</a></li>
    <li><a href="#klipper-mcu">Add EBB as additional MCU to Klipper</a></li>
    <br>
    <li><a href="#klipper-update">Updating</a></li>
  </ul>
  <br>
  <br>
  <br>
  <li><details><summary>Hacky appendix: misuse an EBB as a Klipper USB-to-CAN bridge <i>(click to expand)</i></summary>
  <br>
  <ul>
    <li><a href="#misuse">Introduction - why the heck?</a></li>
    <br>
    <li><a href="#misuse-canboot">CanBoot configuration</a></li>
    <li><a href="#misuse-klipper">Klipper configuration</a></li>
    <li><a href="#misuse-can0">Add can0 interface</a></li>
    <li><a href="#misuse-mcu">Add EBB as additional MCU to Klipper</a></li>
    <br>
    <li><a href="#misuse-update">Update CanBoot/Klipper</a></li>
  </ul>
  </details></li>
</ul>
<br>
<br>
<br>

<hr>

<h2 id="canboot">
  CanBoot bootloader <i>(optional)</i>
</h2>

<br>

<p><a href="#https://github.com/Arksine/CanBoot">Canboot</a> is a bootloader for MCUs to be able to update/flash them via CANBUS.<br>With CanBoot there is no physical intervention (e.g. pressing the boot button) required to flash/update firmware to the MCUs.</p>
<p>Canboot is not mandatory to get klipper running - choose yourself if you like to use it or not.<br><i>If you do not like to use it start at the <a href="#klipper">Klipper</a> section.</i></p>

<br>

<hr>

<h3 id="canboot-download">
  Download Canboot
</h3>

Download CanBoot on your SBC.

```bash
cd ~
git clone https://github.com/Arksine/CanBoot
```

Add CanBoot to your moonraker update manager *(optional)*.

```yaml
[update_manager canboot]
type: git_repo
origin: https://github.com/Arksine/CanBoot.git
path: ~/CanBoot
is_system_service: False
```

<h3 id="canboot-config">
  Configure CanBoot for the EBB
</h3>

Open the config dialog with the following commands:

```bash
cd ~/CanBoot
make menuconfig
```

and use following config settings:
- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32G0B1**
- Build CanBoot deployment application: **8KiB bootloader**
- Clock Reference: **8 MHz crystal**
- Communication interface: **CAN bus (on PB0/PB1)**
- Application start offset: **8KiB offset**
- CAN bus speed: **500000**
- Support bootloader entry on rapid double click of reset button: **check** *(optional but recommend)*
- Enable Status LED: **check**
- Status LED GPIO Pin: **PA13**

this should then look like this:

<p align="center"><img src="./.img/ebb_canboot-make-menuconfig.png"></p>

use `q` for exit and `y` for save these settings.

These lines just clear the cache and compile the CanBoot bootloader.

```bash
make clean
make
```

<br>

<h3 id="canboot-flash">
  Flash the CanBoot bootloader to the EBB
</h3>

> ⚠ Before you start the flashing process, disconnect the heater from the board! \
> Up to version v1.1, the heater output is switched to on in DFU mode while in this mode! \
> This can lead to a fire! 🔥 \
> Like in the next picture where the door bell ringed 🔔 \
> In version v1.2, this pin has changed because of this issue.

<p align="center"><img src="./.img/pre-ebbv1.2-dfu-heating.jpg"></p>

<p align="center"><i>(only the printer was harmed - picture used with permission)</i></p>

<br>

First, you have to put the board into DFU mode. \
To do this, press and hold the boot button and then disconnect and reconnect the power supply, 
or press the reset button on the board. \
With the command `dfu-util -l`, you can check if the board is in DFU mode.

If `dfu-util` can discover a board in DFU mode it should then look like this:

<p align="center"><img src="./.img/ebb_dfu-util_-l.svg"></p>

If this is not the case, repeat the boot/restart process and test it again.

<br>

If your board is in DFU mode, you can flash CanBoot with the following command:

```bash
dfu-util -a 0 -D ~/CanBoot/out/canboot.bin -s 0x08000000:mass-erase:force:leave
```

<p align="center"><img src="./.img/ebb_dfu-util_flash_canboot.svg"></p>

Now press the reset button and if the flash process was successfully one led should blink now.

<br>

<hr style="width:90%">

<br>

<h3 id="canboot-update">
  Update CanBoot ...
</h3>

<br>

... the classic way with an USB cable + <code>dfu-util</code> and without CanBoot

<ul>
  <li>grab a long enough USB cable to connect the EBB and your SBC</li>
  <li>continue if you would <a href="#canboot-flash">flash CanBoot for the first time with <code>dfu-util</code></a></li>
</ul>

<br>

<details><summary>... with the help of CanBoot itself <i>(click to expand)</i></summary>

<br>

Since the board can only be addressed via CAN, further CanBoot updates must also be flashed to the board via CAN. \
This is very easy with the CanBoot bootloader:

```bash
python3 ~/CanBoot/scripts/flash_can.py -f ~/CanBoot/out/canboot.bin -i can0 -u <uuid>
```

<p align="center"><img src="./.img/ebb_canboot_update_canboot.svg"></p>

</details>

<br>

<hr>

<br>

<h2 id="klipper">
  Klipper
</h2>

<br>

<h3 id="klipper-config">
  Configure the Klipper firmware for the EBB
</h3>

Open the config interface of the Klipper firmware with following commands:

```bash
cd ~/klipper
make menuconfig
```

and set the following settings:
- Enable extra low-level configuration options: **check**
- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32G0B1**
- Bootloader offset: **No bootloader** *(without CanBoot)*
- Bootloader offset: **8KiB bootloader** *(with CanBoot)*
- Clock Reference: **8 MHz crystal**
- Communication interface: **CAN bus (on PB0/PB1)**
- CAN bus speed: **500000**

The result should look like this:

<p align="center"><img src="./.img/ebb_klipper-make-menuconfig.png"></p>

use `q` for exit and `y` for save these settings.

Now clear the cache and compile the Klipper firmware:

```bash
make clean
make
```

<h3 id="klipper-flash">
  Flash Klipper ...
</h3>

<br>

<details><summary>... the classic way with an USB cable + <code>dfu-util</code> and without CanBoot <i>(click to expand)</i></summary>

<br>

> Keep in mind: \
> ⚠ Before you start the flashing process, disconnect the heater from the board! \
> Up to version v1.1, the heater output is switched to on in DFU mode while in this mode! \
> This can lead to a fire! 🔥 \
> Like in the next picture where the door bell ringed 🔔

<p align="center"><img src="./.img/pre-ebbv1.2-dfu-heating.jpg"></p>

<p align="center"><i>(only the printer was harmed - picture used with permission)</i></p>

<br>

First, you have to put the board into DFU mode. \
To do this, press and hold the boot button and then disconnect and reconnect the power supply, 
or press the reset button on the board. \
With the command `dfu-util -l`, you can check if the board is in DFU mode.

It should then look like this:

<p align="center"><img src="./.img/ebb_dfu-util_-l.svg"></p>

If this is not the case, repeat the boot/restart process and test it again.

<br>

If your board is in DFU mode, you can flash it with the following command:

```bash
dfu-util -a 0 -D ~/klipper/out/klipper.bin -s 0x08000000:mass-erase:force:leave
```

<p align="center"><img src="./.img/ebb_dfu-util_flash_klipper.svg"></p>

<br>

<hr style="width:90%">

<br>

</details><!-- end of "flash klipper - dfu-util" -->

<br>

<details><summary>... with the help of CanBoot <i>(click to expand)</i></summary>

<br>

> The EBB must be in bootloader mode. \
> The status led should blink in bootloader mode. \
> If not double press the reset button to enter the bootloader

Find the UUID of your EBB:

```bash
python3 ~/CanBoot/scripts/flash_can.py -i can0 -q
```

The output should look like this:

<p align="center"><img src="./.img/canboot_query_can.svg"></p>

With the UUID you have just read, you can now flash the board with:

```bash
python3 ~/CanBoot/scripts/flash_can.py -f ~/klipper/out/klipper.bin -i can0 -u <uuid>
```

<p align="center"><img src="./.img/ebb_canboot_flash_klipper.svg"></p>

<br>

<hr style="width:90%">

</details><!-- end of "flash Klipper - CanBoot" -->

<br>

<h3 id="klipper-mcu">
  Add the EBB as a MCU to Klipper
</h3>

And finally you can add the EBB to your Klipper `printer.cfg` with its UUID - <i>(discover instructions below)</i>.

```yaml
[mcu EBB]
canbus_uuid: <uuid>

# embedded temperature sensor
[temperature_sensor EBB]
sensor_type: temperature_mcu
sensor_mcu: EBB
min_temp: 0
max_temp: 100
```

If you do not use CanBoot and want to find the UUID of your EBB use this command:

```bash
~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
```

The output should look like this:

<p align="center"><img src="./.img/klipper_query_can.svg"></p>

<p><i>CanBus users have already discovered their UUID at the <a href="#klipper-flash">Flash Klipper ... with the help of CanBoot</a> section</i></p>

<br>

<hr>

<h3 id="klipper-update">
  Update Klipper ...
</h3>

<br>

... the classic way with an USB cable + <code>dfu-util</code> and without CanBoot

<ul>
  <li>grab a long enough USB cable to connect the EBB and your SBC</li>
  <li>continue if you would <a href="#klipper-flash">flash Klipper for the first time with <code>dfu-util</code></a></li>
</ul>

<br>

<details><summary>... with the help of CanBoot <i>(click to expand)</i></summary>

<br>

Since the board can only be addressed via CAN, Klipper must also be flashed to the board via CAN. \
This is very easy with the CanBoot bootloader:

```bash
python3 ~/CanBoot/scripts/flash_can.py -f ~/klipper/out/klipper.bin -i can0 -u <uuid>
```

<p align="center"><img src="./.img/ebb_canboot_flash_klipper.svg"></p>

</details>

<br>

<hr>

<br>

<h2 id="misuse">
  Hacky appendix: Misuse an EBB as a Klipper USB-to-CAN bridge
</h2>

<details><summary><i>Click here to expand this section</i></summary>

<br>

When [I (docgalaxyblock)](https://github.com/docgalaxyblock) wanted to place my order the U2C was sold out. \
So I ordered two EBB(36) and misuse one of them as a USB-to-CAN interface.

*A small benefit is that I now have some additional ports - I would not have them if I bought a U2C*

If you also like to misuse one of your EBB as a USB-to-CAN follow the upper guide but just use these CanBoot and Klipper configurations:

<br>

<h3 id="misuse-canboot">
  EBB USB-to-CAN CanBoot
</h3>

<br>

<p>head over to the <a href="#canboot-config">normal CanBoot configuration</a> but use following settings instead:</p>

- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32G0B1**
- Build CanBoot deployment application: **8KiB bootloader**
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB (on PA11/PA12)**
- Support bootloader entry on rapid double click of reset button: **check** *(optional but recommend)*
- Enable Status LED: **check**
- Status LED GPIO Pin: **PA13**

this should then look like this:

<p align="center"><img src="./.img/ebb_misuse_canboot-make-menuconfig.png"></p>

<hr style="width:90%">

<h3 id="misuse-klipper">
  EBB USB-to-CAN Klipper
</h3>

<br>

<p>head over to the <a href="#klipper-config">normal Klipper configuration</a> but use the following settings instead:</p>

- Enable extra low-level configuration options: **check**
- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32G0B1**
- Bootloader offset: **No bootloader** *(without CanBoot)*
- Bootloader offset: **8KiB bootloader** *(with CanBoot)*
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB to CAN bus bridge (USB on PA11/PA12)**
- CAN bus interface: **CAN bus (on PB0/PB1)**
- Application start offset: **8KiB offset**
- CAN bus speed: **500000**

The result should look like this:

<p align="center"><img src="./.img/ebb_misuse_klipper-make-menuconfig.png"></p>

To flash Klipper ...

<br>

<p>... the classic way with an USB cable + <code>dfu-util</code> and without CanBoot: head over to the <a href="#--flash-klipper-">normal Klipper dfu-flashing method</a></p>

<br>

<details><summary>... with the help of CanBoot <i>(click to expand)</i></summary>

<br>

> The EBB must be in bootloader mode. \
> The status led should blink in bootloader mode. \
> If not double press the reset button to enter the bootloader

```bash
# find your <serial device> full path
ls /dev/serial/by-id/*
# the result should be `/dev/serial/by-id/usb-CanBoot_stm32g0b1...`
#
# and flash klipper to it
python3 ~/CanBoot/scripts/flash_can.py -f ~/klipper/out/klipper.bin -d <serial device>
```

<p align="center"><img src="./.img/ebb_misuse_canboot_flash_klipper.svg"></p>

<br>

</details><!-- end of "misuse canboot flash klipper" -->

<hr style="width:90%">

<h3 id="misuse-can0">
  Add can0 interface in mainsailOS
</h3>

Now you only have to create the interface in the OS. \
To do this, create the file `/etc/network/interfaces.d/can0` and fill it with the following content.

```bash
# open file with nano
sudo nano /etc/network/interfaces.d/can0
```

Content of `/etc/network/interfaces.d/can0`:

```
allow-hotplug can0
iface can0 can static
    bitrate 500000
    up ifconfig $IFACE txqueuelen 128
```

> The first line must be different here. Instead of `auto can0`, we use `allow-hotplug can0` because Klipper can
> restart the MCU, and the USB-to-CAN adapter is also restarted. \
> Thus the OS can reactivate the interface automatically.

To save and close the nano editor:  
`ctrl+s` => save file  
`ctrl+x` => close editor

After a reboot, the can0 interface should be ready.

<h3 id="misuse-mcu">
  Add EBB as additional MCU to Klipper
</h3>

Now add the EBB to your Klipper `printer.cfg` with its UUID - *(discover instructions below)*.

```yaml
[mcu EBB_USB-CAN]
canbus_uuid: <uuid>

# embedded temperature sensor
[temperature_sensor EBB_U-C]
sensor_type: temperature_mcu
sensor_mcu: EBB_USB-CAN
min_temp: 0
max_temp: 100
```

If you do not use CanBoot and want to find the UUID of your EBB use this command:

```bash
~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
```

The output should look like this:

<p align="center"><img src="./.img/klipper_query_can.svg"></p>

<p><i>CanBoot users have already discovered their UUID at the <a href="#klipper-flash">Flash Klipper ... with the help of CanBoot</a> section</i></p>

<h3 id="misuse-update">
  Update ...
</h3>

<br>

... the classic way with an USB cable + <code>dfu-util</code> and without CanBoot

<ul>
  <li>continue if you would <a href="#misuse-canboot">flash CanBoot for the first time with <code>dfu-util</code></a></li>
  <li>continue if you would <a href="#misuse-klipper">flash Klipper for the first time with <code>dfu-util</code></a></li>
</ul>

<br>

<details><summary>... with the help of CanBoot itself <i>(click to expand)</i></summary>

<br>

Since the board can only be addressed via CAN, further CanBoot/Klipper updates must also be flashed to the board via CAN. \
This is very easy with the CanBoot bootloader:

```bash
# restart the mcu into CanBoot
python3 ~/CanBoot/scripts/flash_can.py -r -i can0 -u <uuid>
# the status led should blink now in bootloader mode
#
# and...
# ... to update CanBoot
python3 ~/CanBoot/scripts/flash_can.py -f ~/CanBoot/out/canboot.bin -d <serial device>
# ... to update Klipper
python3 ~/CanBoot/scripts/flash_can.py -f ~/klipper/out/klipper.bin -d <serial device>
```

<p align="center"><img src="./.img/ebb_misuse_canboot_update_canboot.svg"></p>
<p align="center"><img src="./.img/ebb_misuse_canboot_update_klipper.svg"></p>

</details><!-- end of "Misuse updating..." -->

</details><!-- end of "Misuse an EBB..." -->
