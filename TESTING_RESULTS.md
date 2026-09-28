# Test results

What has actually been verified, and on what. The point of this file is to be
specific enough that a reader can tell which claims are measured and which are
assumptions from a datasheet nobody in the project has.

## Automated checks

`make check` builds the module and runs `tests/smoke.sh`. The build and static
portions run anywhere. The hardware portion needs root and a machine with the
chip, and is skipped rather than failed without one, so this is usable on a
build machine as a plain build test.

```
15 passed, 0 failed, 0 skipped
```

Covered:

- the module compiles for the running kernel with no compiler diagnostics
- the built module's vermagic matches the running kernel
- the temperature read is cast to `s8` and no ineffective `(char)` cast remains
- `store_pwm` propagates a failed unlock handshake rather than reporting success
- all 7 temperature, 14 voltage, 8 fan and 8 pwm channels are present
- a PWM write is accepted, read back, and the original value is restored
- the fan is still turning afterwards

The static checks exist because two of the bugs below are invisible to a
runtime test on the machine that has the hardware: the module that misreported
193 °C was the module that built and loaded cleanly, and the code that
discarded a PWM write reported success on every run. A check that only runs
against live hardware would have passed both.

## Measured on a real board

**MSI MAG X870E Tomahawk WIFI, Nuvoton NCT6687D, EC firmware 0.0, Linux
7.2.2-arch1-1, GCC 16.2.1.**

### Temperatures read as signed

`temp6` "PCIe x1" reported `+193.0 °C` on a board sitting at 30 °C.

Cause: kbuild compiles modules with `-funsigned-char`, so `char` is unsigned and
a cast to it is a no-op. The driver wrote

```c
s32 value = (char)nct6687_read(data, NCT6687_REG_TEMP(i));
```

which sign-extends nothing, so the register byte `0xC1` was read back
zero-extended and multiplied by 1000. The sensor is two's complement, so `0xC1`
is -63.0 °C, which is what the cast was trying to express.

Confirmed in the disassembly: the sequence changed from `movzwl` + `imul` to
`movsbl` + `imul`. After the fix the same channel reads `-63000` mC.

### Per-channel config registers do not identify absent channels

`NCT6687_REG_MON_CFG`, `NCT6687_REG_FANIN_CFG` and `NCT6687_REG_FANOUT_CFG`
are defined in the driver and were never read. The idea was that a board which
does not populate a channel would leave it disabled there, letting sysfs
exposure follow the hardware instead of exposing all 37 channels
unconditionally with a plausible label each.

Raw values on this board:

```
MON_CFG(0..13)   = 20 02 09 31 0a 50 00 00 00 00 00 00 00 00
FANIN_CFG(0..7)  = 00 x8
FANOUT_CFG(0..7) = a0 a2 aa ac ae b0 b2 a4
```

This does not support the inference:

| Channel | MON_CFG | Reading | Populated |
|---|---|---|---|
| CPU 1P8 | 0x00 | 0 mV | no |
| CPU VDDP | 0x00 | 0 mV | no |
| +3.3V | 0x00 | 3356 mV | yes |
| CPU SA | 0x00 | 898 mV | yes |
| Voltage #2 | 0x00 | 1522 mV | yes |
| AVCC3 | 0x00 | 3240 mV | yes |
| AVSB | 0x00 | 3356 mV | yes |
| VBat | 0x00 | 1060 mV | yes |

A cleared bit does not mean an unpopulated channel. Hiding on one would have
removed seven working voltage sensors and kept both dead ones.

`FANIN_CFG` is zero for all eight inputs while three fans spin, so it carries no
presence information at all. `FANOUT_CFG` reads as a PWM or scaling table rather
than an enable map, and the driver's existing comment already records that
these offsets read as zero on boards where they do not apply. There is also no
established correspondence between the index used by `NCT6687_REG_VOLTAGE()`
and the `MON_CFG` byte that would govern it, since the voltage table's `.reg`
field is an independent permutation.

The registers are now read and logged under `pr_debug` so a board maintainer can
add evidence, and **no channel is hidden on the strength of them**. What would
make a presence filter workable is the datasheet; without it, guessing which bit
means what has the failure mode of a working sensor disappearing.

### PWM write is applied and reported

```
echo 200 > /sys/class/hwmon/hwmon12/pwm1   -> accepted, reads back 200
echo 300 > /sys/class/hwmon/hwmon12/pwm1   -> Invalid argument
```

Fan RPM responded to the change, and the original duty was restored afterwards.

## Not verified

- **No other board.** Everything above is one MSI X870E. The DMI board list
  covers several MSI models; the `msi_alt` fan register mapping is exercised
  only if the DMI match fires.
- **No other kernel.** Builds are verified in CI against 5.15 through 6.17 and
  mainline. Runtime behaviour is verified only on 7.2.2.
- **The `start_fan_cfg_update` timeout path** propagates `-ETIMEDOUT` now, but
  has not been observed to fire. Triggering it requires an EC that stops
  responding mid-handshake, which is not reproducible on demand.
- **`NCT6687_HWM_CFG` read back as `0x80`** on this board, which is the expected
  value. The `0xFF` case is logged, not guarded, and untested.
- **Suspend/resume** restores the cached `hwm_cfg` without re-validating it.
  Not exercised here.

## Rejected approaches

Recorded so they are not re-attempted without new information.

**Filtering on implausible values in the kernel.** A reading that looks wrong is
presentation policy. It belongs in userspace, where it is reversible and cannot
cost a genuinely overheating sensor its reading. The kernel module reports the
hardware and the best-known hardware presence state, and says so when that
state is unknown.

**Treating a register readback as an error signal.** `inb_p()` returns the byte
read from the I/O port and carries no independent status, so a read cannot be
distinguished from a stuck bus, and `0xFF` may simply be the data. No error is
inferred from a register value anywhere in this driver. The only real failure
evidence is the fan configuration handshake timing out, which is why those paths
propagate an error.

**Guarding `NCT6687_HWM_CFG` against `0xFF`.** Only bit 7 of that register is
defined by this driver, so every other bit pattern is legal and `0xFF` cannot be
shown to be invalid. It is logged instead, so that someone on an affected board
has something to report.
