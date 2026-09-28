![https://valid.x86.fr/vsb4yv](https://valid.x86.fr/cache/banner/vsb4yv-6.png)
![https://valid.x86.fr/20aiek](https://valid.x86.fr/cache/banner/20aiek-6.png)
# NCT6687D Kernel module

This kernel module permit to recognize the chipset Nuvoton NCT6687-R in lm-sensors package.
This sensor is present on some B550 motherboard such as MSI or ASUS.

The implementation is minimalist and was done by reverse coding of Windows 10 source code from [LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor)
<br><br>

## Installation
### via package manager
#### .deb package
- Clone this repository
```shell
~$ git clone https://github.com/Fred78290/nct6687d
~$ cd nct6687d
```
- Build package
```shell
~$ make deb
```
- Install package
```shell
~$ dpkg -i ../nct6687d-dkms_*.deb
```
<br>

#### .rpm package (akmod)
- Clone this repository
```shell
~$ git clone https://github.com/Fred78290/nct6687d
~$ cd nct6687d
```
- Build & install package
```shell
~$ make akmod
```
<br><br>

### Manual Install
#### Dependencies:
- Ubuntu/Debian:
	 ```apt-get install build-essential linux-headers-$(uname -r) dkms dh-dkms```
- Fedora/CentOS/RHEL:
	```yum install make automake gcc gcc-c++ kernel-devel kernel-headers dkms```
- ArchLinux:
	 ```pacman -S make automake linux-firmware linux-headers dkms base-devel```
- openSUSE:
	 ```zypper in git make gcc dkms```
<br>

#### Build with DKMS
```shell
~$ git clone https://github.com/Fred78290/nct6687d
~$ cd nct6687d
~$ make dkms/install
```
<br>

#### Manual build
```shell
~$ git clone (this-repo)
~$ cd nct6687d
~$ make install
```
<br>

## What this driver exposes

Every channel is registered in sysfs unconditionally, with a fixed label:

| Group | Count | Files |
|---|---|---|
| Temperature | 7 | `temp1_input` .. `temp7_input` |
| Voltage | 14 | `in0_input` .. `in13_input` |
| Fan | 8 | `fan1_input` .. `fan8_input` |
| PWM | 8 | `pwm1` .. `pwm8`, `pwmN_enable` |

This is the same for every chip the driver handles, including the Nct6686d,
which is detected and gets an hwmon device but no additional channels. There is
no current, power or energy support.

`tempN_min` and `tempN_max` are the lowest and highest values seen since the
module loaded. They are not hardware limits and are not writable.

### A board that does not populate every channel

Because exposure is unconditional, a channel the board does not wire up still
appears, with a plausible label and a value that is a consequence of nothing
being connected: an unwired voltage input reads 0 mV, an unwired fan header
reads 0 RPM, an unused temperature register reads 0.

The per-channel configuration registers were expected to be the hardware's own
record of which channels exist, so that exposure could follow the hardware
instead of a per-board file. Measured on an MSI X870E Tomahawk they do not
support that: channels reading correctly share `MON_CFG == 0x00` with the ones
that are dead, so hiding on a cleared bit would remove working sensors. They
are read and logged under `pr_debug` for anyone with the datasheet to build on,
and nothing is hidden on the strength of them. The full evidence is in
`TESTING_RESULTS.md`.

Until then the per-board workaround is an `sensors.d` config with an `ignore`
line per phantom channel, as in `sensors.d/`. This driver deliberately does not
decide on its own that a reading is implausible, because that judgement belongs
in userspace where it is reversible and cannot cost a genuinely overheating
sensor its reading.

## Sensors

Real output from `sensors` on an MSI X870E Tomahawk. The channels reading 0
(`CPU 1P8`, `CPU VDDP`) and the 0 RPM headers are not populated on that board;
see the section above. `PCIe x1` reads -63.0 °C, which is a two's complement
register byte being reported correctly rather than a real temperature.

```
nct6687-isa-0a20
Adapter: ISA adapter
+12V:           12.07 V  (min = +12.02 V, max = +12.07 V)
+5V:             5.02 V  (min =  +5.01 V, max =  +5.02 V)
+3.3V:           3.36 V  (min =  +3.36 V, max =  +3.36 V)
CPU Soc:         1.40 V  (min =  +1.40 V, max =  +1.45 V)
CPU Vcore:     560.00 mV (min =  +0.56 V, max =  +0.56 V)
CPU 1P8:         0.00 V  (min =  +0.00 V, max =  +0.00 V)
CPU VDDP:        0.00 V  (min =  +0.00 V, max =  +0.00 V)
DRAM:            3.41 V  (min =  +3.40 V, max =  +3.41 V)
Chipset:       294.00 mV (min =  +0.28 V, max =  +0.30 V)
CPU SA:        898.00 mV (min =  +0.90 V, max =  +0.90 V)
Voltage #2:      1.52 V  (min =  +1.52 V, max =  +1.53 V)
AVCC3:           3.26 V  (min =  +3.25 V, max =  +3.26 V)
AVSB:            3.36 V  (min =  +3.36 V, max =  +3.36 V)
VBat:            1.06 V  (min =  +1.06 V, max =  +1.06 V)
CPU Fan:       3000 RPM  (min = 2941 RPM, max = 3370 RPM)
Pump Fan:         0 RPM  (min =    0 RPM, max =    0 RPM)
System Fan #1:    0 RPM  (min =    0 RPM, max =    0 RPM)
System Fan #2:    0 RPM  (min =    0 RPM, max =    0 RPM)
System Fan #3:  980 RPM  (min =  980 RPM, max = 1150 RPM)
System Fan #4:    0 RPM  (min =    0 RPM, max =    0 RPM)
System Fan #5:  995 RPM  (min =  995 RPM, max = 1201 RPM)
System Fan #6:    0 RPM  (min =    0 RPM, max =    0 RPM)
CPU:            +47.5°C  (low  = +47.5°C, high = +62.0°C)
System:         +48.5°C  (low  = +48.0°C, high = +49.0°C)
VRM MOS:        +44.5°C  (low  = +44.0°C, high = +44.5°C)
PCH:            +61.0°C  (low  = +60.0°C, high = +61.0°C)
CPU Socket:     +40.0°C  (low  = +40.0°C, high = +41.0°C)
PCIe x1:        -63.0°C  (low  = -63.0°C, high = -63.0°C)
```

<br>

## Load(prob) Sensors on boot

To make it loaded after system boots

Just add nct6687 into /etc/modules

`sudo sh -c 'echo "nct6687" >> /etc/modules'`

### Arch-Linux with systemd

[See entry in the Arch Wiki](https://wiki.archlinux.org/title/Lm_sensors#MSI_MAG_B650_/_Z890_TOMAHAWK_WIFI_(MS-7D75/MS-7E32)_/_MAG_B550_MORTAR_WIFI_(MS-7C94))


`sudo sh -c 'echo "nct6687" >> /etc/modules-load.d/nct6687.conf'`

<br>

## Gnome sensors extensions

![Fan](./images/fan.png) ![Voltage](./images/voltage.png)

<br>

## Testing

```shell
make check
```

Builds the module and runs the smoke tests. The build and static checks run
anywhere; the hardware checks need root and a machine with the chip, and are
skipped rather than failed without one.

Kernel API compatibility is covered by CI, which builds against 5.15, 6.5, 6.8,
6.11, 6.12, 6.14, 6.17 and mainline, with both sides of the two version
boundaries the driver carries (6.11 for `platform_driver.remove`, 6.13 for the
`hwmon-sysfs.h` split). A dkms install and remove round-trip is included.

See `TESTING_RESULTS.md` for what has been measured on hardware, what has not,
and the approaches that were tried and rejected.

## Tested

This module was tested on Ubuntu 20.04 with all kernel available on motherboard [MAG-B550-TOMAHAWK](https://www.msi.com//Motherboard/MAG-B550-TOMAHAWK) running an [AMD 3900X/AMD 5900X](https://www.amd.com/en/products/cpu/amd-ryzen-9-3900x), and on RL8(RHEL8) [MAG-B550M-MORTAR](https://www.msi.com/Motherboard/MAG-B550M-MORTAR) running an [AMD 5700G](https://www.amd.com/en/products/apu/amd-ryzen-7-5700g)

<br>

## Other motherboard supported
- Many people have reported compatibility with MB having Intel H410M & H510M chipset from some manufacturer. See [issue](https://github.com/Fred78290/nct6687d/issues) report.
<br>

## CHANGELOG

- Add support for MSI B460M Bazooka having NCT6687 with another device ID
- Add support to use generic voltage input without multiplier, allows sensors custom conf
- Support giving fan control back to the firmware
<br>

## VOLTAGE MANUAL CONFIGURATION

Some people report that voltage are wrong. The reason is with some motherboard, voltage sensors are not connected on the same nct6687 register.

As example the **VCore** sensor is connected on the **5th** register for AMD but is connected on the **3rd** register for INTEL.
<br>
Also the **DIMM** sensor is connected on the **4th** register for AMD but connected to **5th** register for INTEL.

To allow customize voltage configuration you must add **manual=1** parameter passed to the module at load

`sudo sh -c 'echo "nct6687 manual=1" >> /etc/modules'`

And use a sensors conf like this **/etc/sensors.d/B460M-7C83.conf**

```
# Micro-Star International Co., Ltd.
# MAG B460M BAZOOKA (MS-7C83)

chip "nct6687-*"
    label in0         "+12V"
    label in1         "+5V"
    label in2         "VCore"
    label in3         "Voltage #1"
    label in4         "DIMM"
    label in5         "CPU I/O"
    label in6         "CPU SA"
    label in7         "Voltage #2"
    label in8         "+3.3V"
    label in9         "VTT"
    label in10        "VRef"
    label in11        "VSB"
    label in12        "AVSB"
    label in13        "VBat"

    ignore in3
    ignore in7
    ignore in9
    ignore in10
    ignore in13

    ignore temp6
    ignore temp7

    compute in0       (@ * 12), (@ / 12)
    compute in1       (@ * 5), (@ / 5)
    compute in4       (@ * 2), (@ / 2)
```

## MODULE PARAMETERS

- **force** (bool) (default: false)
  Set to enable support for unknown vendors.

- **manual** (bool) (default: false)
  Set voltage input and voltage label configured with external sensors file.
  You can use custom labels and ignore inputs without setting this option if
  you can figure out their names (see which `*_label` contains builtin label).

- **fan_config** (`default`, `msi_alt1`) (default: `default`) Changes fan configuration 
  for RPM & PWM registers for motherboards with alternative chip configurations.
  
  Motherboards which need this will show CPU/Pump RPMs but no data on System fans.

  `msi_alt1` is automatically enabled for supported MSI motherboards. Manual configuration is only needed for unsupported boards.

- **msi_fan_brute_force** (bool) (default: false) **[BETA]**
  For MSI motherboards with `msi_alt1` configuration: When enabled, writes PWM values to all 7 fan 
  curve control points. This may help with fan control on some MSI boards where standard PWM writes 
  don't take effect immediately. Only affects system fans controlled by the BIOS. Not the CPU fan or pump fan. 
  
  This implementation is based on register mappings from [LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor).
  
  Usage: `modprobe nct6687 msi_fan_brute_force=1`

  Note: This option requires blacklisting the `nct6683` module to prevent it from loading instead of `nct6687`. See the [Issues](#issues) section for detailed instructions.

<details>
  <summary>Supported MSI boards for ("msi_alt1" and "msi_fan_brute_force")</summary>

  | Board | DMI |
  | --- | ---: |
  | PRO B840-P WIFI | MS-7E57 |
  | B840M GAMING PLUS WIFI6E | MS-7E77 |
  | B850 GAMING PLUS WIFI | MS-7E56 |
  | B850 GAMING PLUS WIFI6E | MS-7E80 |
  | B850M GAMING PLUS WIFI6E | MS-7E81 |
  | PRO B850-P WIFI | MS-7E56 |
  | PRO B850M-A WIFI | MS-7E66 |
  | PRO B850M-P WIFI | MS-7E71 |
  | MAG B850M MORTAR WIFI | MS-7E61 |
  | MAG B850 TOMAHAWK MAX WIFI | MS-7E62 |
  | MPG B850 EDGE TI WIFI | MS-7E62 |
  | MPG B850I EDGE TI WIFI | MS-7E79 |
  | B850MPOWER | MS-7E83 |
  | PRO B850-S WIFI6E | MS-7E80 |
  | X870 GAMING PLUS WIFI | MS-7E47 |
  | X870E GAMING PLUS WIFI | MS-7E70 |
  | MAG X870 TOMAHAWK WIFI | MS-7E51 |
  | PRO X870-P WIFI | MS-7E47 |
  | PRO X870E-P WIFI | MS-7E70 |
  | MAG X870E TOMAHAWK WIFI | MS-7E59 |
  | MPG X870E CARBON WIFI | MS-7E49 |
  | MPG X870E EDGE TI WIFI | MS-7E59 |
  | MEG X870E GODLIKE | MS-7E48 |
  | MEG Z890 ACE | MS-7E22 |
  | MEG Z890M ACE | MS-7E23 |
  | MPG Z890 CARBON WIFI | MS-7E17 |
  | MPG Z890M CARBON WIFI | MS-7E18 |
  | MPG Z890 EDGE TI WIFI | MS-7E19 |
  | MPG Z890I EDGE TI WIFI | MS-7E33 |
  | Z890 GAMING PLUS WIFI | MS-7E34 |
  | MAG Z890 TOMAHAWK WIFI | MS-7E32 |
  | PRO Z890-A WIFI | MS-7E32 |
  | PRO Z890-P WIFI | MS-7E34 |
  | PRO Z890-S WIFI | MS-7E54 |
</details>

## CONFIGURATION VIA SYSFS

In order to be able to use this interface you need to know the path as which
it's published. The path isn't fixed and depends on the order in which chips are
registered by the kernel. One way to find it is by device class (`hwmon`) via a
simple command like this:
```
for d in /sys/class/hwmon/*; do echo "$d: $(cat "$d/name")"; done | grep nct6687
```

Possible output:
```
/sys/class/hwmon/hwmon5: nct6687
```

This means that your base path for examples below is `/sys/class/hwmon/hwmon5`
(note that adding/removing hardware can change the path, drop `grep` from the
command above to see all sensors and their relative ordering).

Another way to look it up is by a device (class path actually just points to
device path) like in:

`cd /sys/devices/platform/nct6687.*/hwmon/hwmon*`

The first asterisk will be expanded to an address (`2592` which is `0xa20` that
you can see in `sensors` output) and the second one to a number like `5` from
above.

### `pwm[1-8]`

Gets/sets PWM duty cycle or DC value that defines fan speed.  Which unit is used
depends on what was configured by firmware.

Accepted values: `0`-`255` (slowest to full speed).

Writing to this file changes fan control to manual mode.

Example:

```
# slow down a fan as much as possible (will stop it if the fan supports zero RPM mode)
echo 0 > pwm6
# fix a fan at around half its speed (actual behaviour depends on the fan)
echo 128 > pwm6
# full speed
echo 255 > pwm6
```

### `pwm[1-8]_enable`

Gets/sets controls mode of fan/temperature control.

Accepted values:
 * `1` - manual speed management through `pwm[1-8]`
 * `99` - whatever automatic mode was configured by firmware
          (this is a deliberately weird value to be dropped after adding more
           modes)

Example:

```
# fix a fan at current speed (`echo pwm6` will be constant from now on)
echo 1 > pwm6_enable
# switch back to automatic control set up by firmware (`echo pwm6` is again dynamic after this)
echo 99 > pwm6_enable
# switch to ~25% of max speed
echo 64 > pwm6
# automatic
echo 99 > pwm6_enable
# back to ~25% (it seems to be remembered)
echo 1 > pwm6_enable
```

## VERIFIED
**1. Fan speed control**

- Changing fan speed was tested successfully by users, see in [reported issues](https://github.com/Fred78290/nct6687d/issues).

## Issues
### ACPI
loading nct6687 fails. `journalctl` shows `ACPI: OSL: Resource conflict; ACPI support missing from driver?`:
* add `acpi_enforce_resources=lax` as a kernel parameter

### Loading fails during startup
`dmesg` / `journalctl` shows 
```
kernel: nct6687: EC base I/O port unconfigured
systemd-modules-load[339]: Failed to insert module 'nct6687': No such device
```
* add `softdep nct6687 pre: i2c_i801` to e.g. `/etc/modprobe.d/sensors.conf`.

### Loading `msi_fan_brute_force` parameter fails
- Symptom: No RPM value is displayed for some fans in the sensor data; or the following behavior is observed: the fan stops at 60%
- Solution steps:
  1. Prevent the wrong driver from loading:
     ```sh
     echo "blacklist nct6683" | sudo tee /etc/modprobe.d/nct6683_blacklist.conf
     ```
  2. Set module option to enable brute-force fan curve writes for MSI boards:
     ```sh
     echo "options nct6687 msi_fan_brute_force=1" | sudo tee /etc/modprobe.d/nct6687_msi.conf
     ```
  3. Ensure the `nct6687` module is loaded at boot:
     ```sh
     echo "nct6687" | sudo tee /etc/modules-load.d/nct6687.conf
     ```
  4. Reboot and verify the module loads correctly.   
   

- After installation:
  Check `/etc/conf.d/lm-sensors.conf` (or your distro's equivalent) that `HWMON_MODULES` not includes `nct6683` (e.g., `HWMON_MODULES="coretemp nct6683 ..."`) to ensure the correct driver is loaded at boot.

### Only CPU & Pump show RPM values
On MSI motherboards, `msi_alt1` configuration is automatically detected and enabled.
You can verify this in `dmesg` after loading the module:
```
nct6687 nct6687.2592: Detected MSI board; using alternative fan configuration (msi_alt1)
nct6687 nct6687.2592: active fan config=msi_alt1, SYS_FAN reg_rpm=0x015E/0x015C/0x015A
```

The second line is always printed and shows the EC offsets the driver reads
for SYS_FAN #1/#2/#3. Use it to confirm the right mapping is active.

If you have a non-MSI motherboard with this issue, try using module parameter `fan_config=msi_alt1` manually.

> **Warning — do NOT force `fan_config=msi_alt1` on non-listed MSI boards.**
> Only the NCT6687DR-equipped MSI families (B840/B850/X870/X870E/Z890) are
> compatible with `msi_alt1`. Earlier MSI series with the plain NCT6687D chip
> — including **B650/B660/X670/Z690/Z790** — use the *default* register
> mapping and are auto-detected correctly without any module parameter.
>
> If you previously set `options nct6687 fan_config=msi_alt1` in
> `/etc/modprobe.d/` on one of those boards as a troubleshooting attempt,
> remove it: forcing `msi_alt1` makes the driver read EC offsets
> `0x154-0x15E` which are zero on non-DR variants, causing all SYS_FAN to
> report `0 RPM` while CPU_FAN keeps working. The `active fan config=...`
> dmesg line above will reveal a stale forced setting at a glance.
>
> Reference: issue #167 (MSI MPG B650 CARBON WIFI, MS-7D74).
