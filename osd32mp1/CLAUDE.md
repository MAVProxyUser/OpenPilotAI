# Working on the OSD32MP1-RED

Rules and traps for the Octavo OSD32MP1-RED bring-up. Each one cost real time
to learn — read before changing something, not after it breaks.

Companion docs: `README.md` (what this is, current state), `SKILLS.md` (the
commands). The flight-code rules live in
`../NinjaPilot-15.02.ninja/CLAUDE.md` and still apply once firmware is running
here.

---

## RULE: the BOARD REVISION picks the DTB, and getting it wrong looks like dead hardware

This board is **hardware revision V1.2**. Octavo's v3.0 image ships only the
**V1.1** device tree. They are not interchangeable — the Ethernet PHY changed:

| rev | PHY | interface | MDIO addr |
|---|---|---|---|
| V1.1 | (RGMII part) | RGMII | 3 |
| **V1.2** | **KSZ8051RNLU** | **RMII** | **7** |

Booting the V1.1 DTB on a V1.2 board gives `no phy at addr -1` and no
Ethernet, while the link LEDs still blink — because the LEDs are driven by the
PHY's own link detection, which works regardless of whether the SoC can talk
to it over MDIO. **Blinking link lights are not evidence the MAC is working.**

The fix was not a cable, a jumper, or a config: it was
`osd32mp1-red-v1_2-trusted-openstlinux-sdcard-v3_0_1.zip`, whose boot
partition carries `stm32mp157c-osd32mp1-red-v1_2.dtb`. Ethernet came up on the
first boot with it.

**The schematic says so outright** — `OSD32MP15x_RED_7x_sch-V1_2`, Rev 1.2
change list, item 6: *"Updated U10 (Ethernet PHY) to KSZ8051RNL."* The part
changed with the board revision, so the device tree had to as well. Item 4 of
the same list (*"Updated U20 (CAN IC) to TJA1441BTK"*) is the CAN transceiver,
for the same reason.

Two wrong theories were held along the way, both worth not repeating:
- "the PHY needs 50 MHz REF_CLK and isn't getting it" — plausible, and wrong;
  the user had physically disconnected the PHY during that test, so the
  evidence never supported it
- "a rescan needs a reboot" — disproved by `no phy at addr -1` appearing at
  ifup time

**Check the silkscreen revision against the DTB filename before debugging
Ethernet at all.**

## The gadget is currently EITHER a network device OR a console — and the cause is dwc2 FIFOs, NOT macOS

Measured 2026-08-15 against the running board:

| gadget functions | `/dev/cu.usbmodem*` | USB network |
|---|---|---|
| `ecm.0` + `acm.0` (either order) | yes | **no** — `AppleUSBECMData=0` |
| `ecm.0` alone | no | yes — host gets 192.168.7.x, board answers on .1 |
| `ncm.0` + `acm.0` | yes | no — both drivers attach, carrier never negotiates |

**CORRECTION.** This was first written up as "macOS will not bind both", which
is WRONG and was believed for several hours because the host-side evidence
(`AppleUSBECMControl=1`, `AppleUSBECMData=0`) is genuinely consistent with it.
The board's own kernel log gives the real answer:

    dwc2 49000000.usb-otg: dwc2_hsotg_ep_enable: No suitable fifo found
      dwc2_hsotg_ep_enable <- usb_ep_enable <- gether_connect [u_ether]
        <- ecm_set_alt [usb_f_ecm] <- composite_setup [libcomposite]

macOS did exactly the right thing — it selected ECM's data alt-setting — and
**the BOARD failed to enable the endpoint**, so no carrier ever came up and
`usb0` stayed at `no-carrier (configuring)`. The host was never the problem.

The mechanism is TX FIFO exhaustion in the dwc2 gadget controller. The device
tree pins the partitioning:

    g-rx-fifo-size     0x200
    g-np-tx-fifo-size  0x20
    g-tx-fifo-size     <0x100 0x10 ...>   one entry per IN endpoint

ECM (bulk IN + interrupt IN) plus ACM (bulk IN + interrupt IN) needs **four**
IN endpoints, and the available FIFO RAM / entry list does not stretch that
far. Adding functions is not free on this controller.

So "either/or" is a **current limitation, not a law** — extending
`g-tx-fifo-size` in the DTB is the fix worth trying if both are ever wanted at
once. Until then the shipped config chooses the console (`acm.0` + `ecm.0`),
because Ethernet already covers networking and the console survives any
network misconfiguration. Switching is one word — see `SKILLS.md`.

**Method lesson:** host-side symptoms told a coherent, wrong story. The board's
`dmesg` had the answer the whole time and was not read until much later. When
two devices disagree, read BOTH logs before concluding which one is at fault.

### Corollary: do NOT read USB descriptors before the image is actually running

Descriptors were dumped after flashing but before the new image had booted,
which produced the confident and wrong claim *"this gadget was never RNDIS."*
The V1.2 OpenSTLinux rootfs ships `func_eth=rndis.0`, and RNDIS is a Microsoft
protocol macOS cannot use at all. A descriptor dump only describes whatever
gadget is bound **right now**.

Related: this image's ACM advertises `bInterfaceProtocol 0x01`, so macOS binds
it natively and creates `/dev/cu.usbmodem*`. The older Debian image advertised
`0xff` and macOS refused it, which is why `usb_console.py` (libusb) exists.
**These two images behave oppositely — do not carry conclusions between them.**

## RULE: edit disk images offline as FILES, and fsck every time

macOS cannot mount ext4 and `/dev/disk*` needs root, but a raw image is just a
file you own, so the whole edit runs unprivileged: parse the GPT for byte
offsets, `dd` the partition out, `debugfs` it, `e2fsck`, inject it back.
`gpt.py` and `part.py` do this; never hardcode offsets.

Two failure modes, both hit for real:

- **`write` + `ln` + `rm` in one debugfs session corrupted the image** — the
  operations collided on one inode (26349). **`cd` into the target directory
  and `write` directly.** No hard-link/remove dance.
- A structural error introduced here does not announce itself; it shows up
  much later as a card that will not boot. **`e2fsck -fy` after editing, and
  again after injecting.** Both must come back with zero repairs.

## dropbear on this image: two gotchas that both read as "SSH is broken"

- **It offers only the SHA-1 `ssh-rsa` host key and pubkey algorithm.**
  OpenSSH 8.8+ disables those by default, so a correct setup still dies with
  `no matching host key type found. Their offer: ssh-rsa`. Both
  `HostKeyAlgorithms=+ssh-rsa` and `PubkeyAcceptedAlgorithms=+ssh-rsa` are
  required, and are in `~/.ssh/config` under `Host osd32mp1`.
- **root ships with an EMPTY password**, and dropbear refuses blank-password
  authentication. `dropbear.socket` being enabled is therefore not sufficient
  to log in. The edited image installs an `authorized_keys` and sets a real
  hash.

Note `iptables.service` and `ip6tables.service` are enabled but
`/etc/iptables/iptables.rules` is **empty** — the firewall was suspected and
is not a factor.

## TRAP: tearing down the gadget at runtime kills the console permanently

`serial-getty@.service` carries `BindsTo=dev-%i.device`. When the USB gadget is
unbound, `/dev/ttyGS0` disappears, systemd treats that as a *clean stop*, and
`Restart=always` therefore never fires. The getty stays dead even after the
device returns.

This only bites when rebuilding the gadget live (as the ECM/ACM experiments
did). A normal boot is fine. Recovery is `systemctl restart
serial-getty@ttyGS0`, or a reboot — and a reboot is usually better, because
repeated live teardowns also wedge the ACM endpoint itself, at which point
even opening `/dev/cu.usbmodem*` hangs with nothing holding the port.

The image also ships a drop-in with `StartLimitIntervalSec=0` and
`RestartSec=2`, because `ttyGS0` does not exist until `usbotg-config.service`
binds at `multi-user.target` — without it systemd burns its default
5-starts-in-10s budget and gives up before the node ever appears.

## RULE: `console=` ORDER on the kernel cmdline is load-bearing

`extlinux.conf` reads:

    console=ttyGS0,115200 console=ttySTM0,115200

ttyGS0 goes **first on purpose**. The *last* `console=` becomes `/dev/console`,
and ttyGS0 only exists while USB is plugged in and the gadget is bound. Making
a disappears-when-unplugged device primary would throw systemd's output away
and cost the UART console too.

## Octavo's eMMC deploy tarball is genuinely truncated (this was challenged, so: proof)

Two independent downloads of `osd32mp1-red-emmc-deploy.tar.xz`:

    size   83886080 bytes  (both)  = exactly 80 MiB
    md5    ae6fbd99fc1bf313836775df042b0dd6  (both, IDENTICAL)
    xz -t  "Unexpected end of input", exit 1

Byte-identical output from two separate transfers rules out a network fault —
network corruption does not reproduce bit-for-bit. xz verifies a CRC64 per
block plus a stream footer, so it is detecting a genuinely absent end of
stream. The bootloaders inside survive because they sit at the front of the
tar; only the rootfs was cut off. This is why the forum recovery procedure
fails for people who follow it.

**The scepticism was still right in principle** — "the vendor's file is
broken" is a claim that should be proven, not asserted, and the proof above is
what a challenge is for.

## The board cannot be bricked

The STM32MP1 ROM bootloader is in silicon. It always offers USB-DFU
(`0483:df11`) when it cannot boot the selected medium, and the ROM has **no
eMMC write path** — DFU phase `0x01 fsbl1-boot` loads TF-A into SYSRAM only. A
power cycle always returns a clean recovery state. Experiment freely.

## Verify claims; a self-test caught a real bug here

The `$6$` password hash was generated by a hand-written SHA-512 crypt
(`sha512crypt.py`) because macOS has neither `crypt(3)` `$6$` support nor
`openssl passwd -6`. The implementation was **wrong on the first attempt** —
Drepper's custom base64 packs the first index as the *most* significant byte,
and encoding it little-endian produces a plausible-looking but invalid hash.
The official spec test vector caught it immediately. `sha512crypt.py` refuses
to emit a hash unless that vector passes; keep it that way.

Same lesson, cheaper: a LAN scan turned up a "new" host at `192.168.0.40` that
looked like the board and was a **printer** (ports 80/443/**631 IPP**). One
port scan separated a plausible story from the truth. The board was
identifiable positively — avahi publishes hostname `osd32mp1-red-v1_2` as
`osd32mp1-red-v12` (mDNS drops the underscore).

## Hardware facts worth not re-deriving

- **armv7l, 2 cores, ~426 MB usable RAM**, kernel 5.10.10, OpenSTLinux 3.1
- **`gcc`/`g++` 9.3.0, `make` 4.3, `python3` 3.8.2 are present**
- **`git` and `rsync` are ABSENT** — ship source as a tar over ssh, and expect
  `version-info.py` to need the stub treatment (`0xNone commit_hash_prefix` is
  the symptom of a missing git worktree)
- `can0` already exists as a SocketCAN interface
- DHCP lease moves (`.167` → `.90` across one reboot) — **always address the
  board by its mDNS name**, never a remembered IP
- power: 5V ≥2A, barrel jack 5.5mm/2.5mm, or USB-C with jumper JP3 at 1-2

## Sensor wiring: the two mistakes that cost hardware, and the one that costs a day

- **5V/GND swapped on the Matek JST-GH-4P is destructive.** CANH/CANL swapped
  is harmless — it simply will not communicate. So verify the power pair with
  a multimeter and treat the signal pair as recoverable.
- The board's CAN connector **JP8** (sheet 7) is `4=NC, 3=CANH, 2=CANL, 1=GND`
  and carries **no power**.
- **There are TWO 5.2 V rails and they are not interchangeable.** The CAN
  transceiver runs on `PMIC_BSTOUT_5P2V`; the RPi and mikroBUS headers run on
  `PMIC_VBUSOTG_5P2V/DET`. The header rail is the **USB-OTG VBUS** output,
  which the PMIC normally enables to *source* VBUS in host mode — and we run
  the port in **device/gadget mode**. So `JP7` pin 10 may simply be dead in our
  configuration. **Measure it, booted as we actually run it, before wiring
  anything to it.** A separate bench 5 V with grounds tied avoids the question
  entirely; CAN needs a common reference, not a shared rail.
- **`R48` (120 Ω CAN termination) is DNP — the board can never terminate
  itself.** A CAN bus needs **exactly two** terminators, one at each physical
  END; middle nodes stay open. The Matek `120R` jumpers are open from the
  factory. Wiring the board as the MIDDLE node (`L431 → board → L4-3100`) makes
  the DNP correct by construction and needs no soldering.
  **Check by measuring CANH–CANL with power off: ~60 Ω is right**, 120 Ω means
  one terminator, 40 Ω means three (a real fault — it overloads the drivers).
  Under-termination is the dangerous case because a short bench harness works
  anyway and only fails at airframe cable lengths.
- **VBAT (header `JP24`) is NOT a supply.** It is the RTC / backup-SRAM domain
  *input*, for a battery or supercap. `JP4` is not a 5 V tap either — it sits
  on `PMIC_VOUT4_3P3V` and is DNP.
- Source of truth for all of the above is the schematic
  `OSD32MP15x_RED_7x_sch-V1_2` (sheets 11–12), not the standard RPi/mikroBUS
  pinouts — Octavo's public web docs do not give pin numbers at all.
- **The frame convention is FRD body → NED world**, per `gazebo_bridge.py`'s
  `Q_FLU2FRD` / `Q_ENU2NED`. Every real sensor must be rotated into FRD before
  injection. This is the single most likely source of sign bugs in the whole
  sensor effort, and a sign error reads as a control bug, not a wiring bug.

## SETTLED: DroneCAN bring-up, and the two things that make a healthy bus look dead

Both Matek nodes are up and publishing (2026-08-15):

    node 124  org.ardupilot.MatekL431-Periph   gnss.Fix2 @ 5 Hz
    node 125  org.ardupilot.MatekL431-GPS      ahrs.MagneticFieldStrength @ 25 Hz

Mag reads `[0.2617, -0.0750, 0.3547]` gauss = **0.447 G / 44.7 µT**, inside
Earth's 25–65 µT range and stable to ±0.03 µT — a real sensor, sanity-checked
by magnitude rather than by "numbers appeared".

Getting there hit two failures that both present as "the bus is dead":

**1. No termination anywhere.** `R48` is DNP on the board and the Matek `120R`
jumpers ship open, so the bus had ZERO terminators. Signature, and it is
distinctive: line activity present, **zero decodable frames at EVERY bitrate**
(1M/500k/250k/125k tried), rx error counter pinned at **127**. A bitrate
mismatch does not look like this — one rate would usually work. After
soldering one terminator: `ERROR-ACTIVE`, `berr-counter tx 0 rx 0`, frames
immediately.

Before blaming the bus, prove the controller with a loopback — it takes
seconds and cleanly separates SoC-side from wire-side:

    ip link set can0 up type can bitrate 1000000 loopback on
    cansend can0 123#DEADBEEF        # candump should echo it back

Also worth knowing: `can0` is `4400e000.can` (FDCAN1); its DT node has only
pinctrl and **no standby GPIO**. The transceiver's `S` pin is on **PG3**
(gpiochip6 line 3), which sits unused as an input reading **0** — i.e. the
TJA1441 is already in normal mode. Standby is NOT a candidate cause here; that
was checked and ruled out.

**2. No dynamic node-ID allocator.** AP_Periph nodes boot anonymous and
broadcast `dynamic_node_id.Allocation` until something grants them an ID —
publishing **nothing else** in the meantime. On an aircraft the flight
controller is the allocator; on this bench nothing was, so the nodes retried
forever and the bus looked alive-but-useless. `dronecan_allocator.py` fills the
role and persists grants in a small DB so IDs survive reboots.

### Anonymous frames use a DIFFERENT CAN-ID layout

This produced a confident wrong answer before it was caught. A normal message
puts a 16-bit type in bits 8–23, but an anonymous frame has no source node ID
to identify the sender, so:

    normal     bits 8-23  = message type id (16 bits)
    anonymous  bits 8-9   = message type id (2 bits - only ids 0..3 exist)
               bits 10-23 = discriminator derived from the unique id

Decoding an anonymous frame with the normal layout yields plausible-looking
garbage — it reported "msg 12537" and "msg 39521" for what were really type 1
(Allocation) from two different senders. **Those two bogus numbers were the
only evidence that a second node existed**, so the misdecode was concealing the
device count as well as the message type. Group anonymous senders by
discriminator, never by node id (they are all 0).

## USB host port works — and `usb33` is a RED HERRING

The USB-A host port enumerates normally:

    usb 3-1: new full-speed USB device number 2 using ohci-platform
    cdc_acm 3-1:1.0: ttyACM0: USB ACM device

**Do not diagnose port power from the `usb33` regulator.** It reads
`state=disabled users=0` even while a device is happily enumerated and
streaming — it is not a usable signal for "is the port powered". Time was
spent suspecting it before a device was simply plugged in. The honest test is
`dmesg`: a connect event appears, or it does not. If it does not, measure VBUS
on the port with a meter rather than reading regulator sysfs.

The Klipper accelerometer identifies as **`1d50:614e`**, manufacturer `Anchor`,
product `Rampon` → `/dev/ttyACM0`. It is **silent until spoken to**, which is
correct: Klipper firmware speaks a custom binary protocol (sync-framed, CRC16),
and the host must first fetch the MCU's compressed *data dictionary* before it
can issue any command. So "no output on open" is health, not a fault — but it
also means reading it needs a real protocol implementation, not a serial read.
That keeps it in its documented role: an independent vibration reference, not
a flight sensor.

## SOLVED: reading the KUSBA (Klipper protocol) — four traps, all silent

`klipper_probe.py` (transport + dictionary) and `klipper_accel.py` (ADXL345
streaming) talk to the KUSBA with no Klipper install. Measured 2026-08-15:
**3190 samples in 4 s at 798 Hz**, `DEVID = 0xE5` on `spi0`.

The device is a **KUSBA v2**: ADXL345 on an RP2040, stock Klipper firmware,
`1d50:614e` / `Anchor` / `Rampon`, dictionary reports `MCU rampon_anchor`,
`CLOCK_FREQ 1000000`, 17 commands.

Every failure mode here looks like "dead device", so they are worth knowing:

- **The MCU's expected sequence number PERSISTS across host connections.** It
  is whatever the previous client left it at, NOT 0. Guess wrong and every
  block is silently ignored with no error — indistinguishable from unplugged.
  **Sweep 0..15 to discover it**, then track the value the MCU echoes in each
  reply. This cost a full debugging round: seq 0 worked, then the next run
  failed because the first run had advanced it to 1.
- **Send a run of sync bytes (0x7E) before the first command.** The MCU
  discards input until a sync boundary, so if its parser is mid-garbage-block
  your first well-formed block is eaten as that block's tail.
- **A zero-length payload is a bare ACK block, not an error.** Assuming at
  least one message raises IndexError on every ack.
- **One block can carry SEVERAL messages back to back.** Decoding only the
  first silently drops data — including sensor samples.

Command IDs are **not fixed**; they are assigned per firmware build and
published in the zlib-compressed JSON dictionary fetched via `identify`
(command id 1, the only guaranteed-constant one). Drive everything from the
dictionary — `klipper_accel.py` builds its codec from it at connect time.

**Accuracy caveat:** magnitude at rest reads **0.912 g**, ~9% low. That is
uncalibrated ADXL345 zero-g offset (spec is ±150 mg/axis typical) plus
sensitivity tolerance, not a decode bug — the axes are stable to ~0.05 m/s²
and the vector direction is consistent. Fine for **relative** vibration work,
which is this device's documented role; it would need calibration before any
absolute use.

## The GPS's IST8310 will NEVER appear - the AP_Periph build has no such driver

Chased as a wiring fault; it is not. ArduPilot's hwdefs for BOTH Matek nodes
(`MatekL431-Periph`, `MatekL431-GPS`) compile in exactly two compass backends:

    COMPASS RM3100   SPI:rm3100  false ROTATION_PITCH_180_YAW_90
    COMPASS QMC5883L I2C:0:0xd   false ROTATION_PITCH_180_YAW_90
    define HAL_COMPASS_MAX_SENSORS 1

So node 124 probes I2C **0x0D** for a QMC5883L. An IST8310 answers at **0x0E**
and has no driver in the image - correctly wired, it still never replies.
`HAL_COMPASS_MAX_SENSORS 1` is why there is a single `COMPASS_DEV_ID` and no
`_DEV_ID2/3`.

Evidence that it is NOT wiring or boot ordering, all measured:
`COMPASS_ENABLE=1`, `COMPASS_DISBLMSK=0`, a full `RestartNode` of node 124 with
everything powered, and still `COMPASS_DEV_ID=0` with zero mag messages - while
node 125's RM3100 publishes 25 Hz throughout. The user confirmed a 6-pin
(UART3+I2C) cable, which killed the wiring theory outright.

**Method lesson, the same one as the ECM/dwc2 case:** the plausible physical
explanation (cable) was pursued ahead of the cheap authoritative one (read the
firmware's hwdef). Check what the software can *possibly* do before theorising
about wires.

**Do not "fix" this.** The RM3100 already on the bus is the better sensor by an
order of magnitude (12.2 nT quantisation measured, vs ~300 nT for an IST8310).
Adding IST8310 support means rebuilding and reflashing AP_Periph for a
redundant, worse compass.

## DONE: fw_simposix builds and runs on OpenSTLinux, fed by REAL sensors

2026-08-15. `fw_simposix.elf` (ARM EABI5, 2.2 MB) built natively on the board
and running under systemd as `fwsimposix.service`, taking sensors ONLY from
`sensor_bridge.py` over UAVTalk/UDP:9000:

    bridge sent  mag +14.7,-19.1,+35.9 uT  -> FW MagSensor  +146.5,-191.2,+359.1 mG
    bridge sent  accel +1.95,+1.07,+8.49   -> FW AccelSensor  +1.99,+0.99,+8.64
                                            -> FW AttitudeState Roll -172.5 Pitch +11.9

The gauss->milligauss conversion checks exactly (14.7 uT = 0.147 G = 147 mG),
and `gyroupdates` was a flat **0** before the bridge started - with
`NINJAPILOT_EXTERNAL_PHYSICS=1` there is no other sensor source, so those
numbers can only have come from the RM3100 on CAN and the ADXL345 on USB.

### The four things the build needed on this image

1. **`package/`** is a separate top-level dir - `Makefile:727` includes
   `package/$(UNAME).mk` and the build dies without it.
2. **`QMAKE=true` PLUS a no-op `build/uavobjgenerator/Makefile`.** There is no
   Qt here. The `uavobjgenerator` target runs qmake *and then* `$(MAKE)` in
   that directory, so stubbing only qmake still fails. UAVObject sources are
   pre-generated elsewhere into `build/uavobject-synthetics` (215 files).
3. **`git`** from the ST feed - `version-info.py` shells out to it, and
   without it you get the `0xNone commit_hash_prefix` failure. A local
   `git init` + one commit is enough.
4. **A non-root build user.** `Makefile:85` refuses to run as root unless
   `FAKEROOTKEY` is set. Creating a `build` user is the honest fix; the check
   exists to stop root-owned artifacts.

### TRAP: `pkill -f fw_simposix` kills your own SSH session

Three launches "silently failed" before this was spotted. `pkill -f` matches
the FULL command line, and an ssh command that launches
`.../fw_simposix.elf` contains that string - so pkill killed the shell that
was about to start the firmware. The `[f]w_simposix` bracket trick does NOT
help, because the binary path in the same command line still matches. Use
`pkill -x fw_simposix.elf` (exact process NAME), or just `systemctl stop`.

Run it under systemd, not `nohup`/`setsid` over ssh - backgrounded processes
did not survive the session:

    systemd-run --unit=fwsimposix --working-directory=/usr/local/ninja/fcwd \
      -E NINJAPILOT_EXTERNAL_PHYSICS=1 \
      /usr/local/ninja/src/build/fw_simposix/fw_simposix.elf

### What this does NOT yet do

`AttitudeState` Roll reads **-172 deg** because the KUSBA is loose on the
bench in an arbitrary orientation - that is the mounting rotation caveat, not
a bug. Yaw barely moves because **GyroSensor is zeros**, and `rateupdates`
stays negative because the inner loop cannot close without a real gyro. Fix
is the ICM-42688-P Click on mikroBUS SPI.

## Do NOT re-litigate: SimPosix already builds and runs here

The port was proven on the earlier Debian image — `fw_simposix.elf` built
natively on armv7l and ran with full PiOS/FreeRTOS init and an LED heartbeat,
EXIT=0. The one genuine portability bug found was in
`flight/pios/inc/pios_posix.h`: `#define false FALSE` sat *outside* the
`#ifndef __cplusplus` guard. macOS hides this because its system headers define
TRUE/FALSE; glibc does not, so it only appears when building for Linux. Nothing
architecture-specific about it.
