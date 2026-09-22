# Linuwu-Sense Fix for Acer Nitro AN515-58

This package contains the patched `Linuwu-Sense` / `DAMX` kernel module tailored for the Acer Nitro AN515-58 with a 4-zone RGB keyboard.

This patch was created based on direct testing on:

* Model: `Nitro AN515-58`
* Kernel during testing: `6.18.34-1-lts`
* Module: `linuwu_sense`
* RGB keyboard: 4-zone, compatible with Jafar/facer pipeline

## Verified Working Features

* Linuwu/DAMX fan control remains functional.
* Linuwu/DAMX battery limiter/control remains functional.
* Static per-zone RGB works.
* Breathing mode RGB works.
* Dynamic RGB payload adapted to Jafar's implementation.
* Power profiles (`quiet`, `balanced`, and `balanced-performance`) no longer revert unexpectedly to `balanced`.

## Key Changes

This patch adds/fixes:

* Dedicated quirk for `AN515-58`.
* 4-zone static RGB using WMI method `6` with the payload `{zone, red, green, blue}`.
* Enabling all zones using `SET_GAMING_LED` instead of `GET_GAMING_LED`.
* Static mode activation using a 16-byte payload to method `20` similar to `facer_rgb.py`.
* Dynamic mode payload aligned with Jafar's implementation for the AN515-58.
* Breathing mode no longer forces `speed=0` on the AN515-58.
* AC power detection uses the Linux power supply API first, avoiding incorrect readings from WMI `BAT_STATUS` on the AN515-58.

## Installation

Run from this folder:

```bash
make
sudo make install

```

Or use the install + reload script:

```bash
sudo ./scripts/install-and-reload.sh

```

If the old module is still active and the patch changes are not detected, reload manually:

```bash
sudo systemctl stop linuwu_sense.service 2>/dev/null || true
sudo rmmod linuwu_sense 2>/dev/null || true
sudo modprobe linuwu_sense
sudo systemctl start linuwu_sense.service 2>/dev/null || true

```

Verify that the runtime version matches the module file:

```bash
cat /sys/module/linuwu_sense/srcversion
modinfo /lib/modules/$(uname -r)/kernel/drivers/platform/x86/linuwu_sense.ko | grep '^srcversion'

```

## RGB Testing

Set 4-zone static colors:

```bash
sudo ./scripts/test-rgb-static.sh

```

Expected output:

* Zone 1: Red
* Zone 2: Green
* Zone 3: Blue
* Zone 4: White

Test magenta breathing mode:

```bash
sudo ./scripts/test-rgb-breathing.sh

```

Manual commands:

```bash
BASE=/sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi
echo ff0000,00ff00,0000ff,ffffff,100 | sudo tee "$BASE/four_zoned_kb/per_zone_mode"
echo 1,4,100,0,255,0,255 | sudo tee "$BASE/four_zoned_kb/four_zone_mode"

```

## Power Profile Testing

Available modes on this AN515-58 model:

```text
quiet
balanced
balanced-performance

```

Check:

```bash
cat /sys/firmware/acpi/platform_profile_choices
cat /sys/firmware/acpi/platform_profile

```

Set profile:

```bash
sudo ./scripts/set-power-profile.sh quiet
sudo ./scripts/set-power-profile.sh balanced
sudo ./scripts/set-power-profile.sh balanced-performance

```

Note: If the DAMX UI has a `Performance` label, map it to `balanced-performance`, not `performance`.

## Troubleshooting

If RGB does not appear:

```bash
ls /sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi/four_zoned_kb
dmesg | grep -iE 'linuwu|acer|wmi|rgb|keyboard|error|fail'

```

If the power profile reverts to `balanced`, check the AC status:

```bash
cat /sys/class/power_supply/ACAD/online

```

It should be `1` when the charger is plugged in.

If the module fails to load due to conflicts:

```bash
lsmod | grep -E 'linuwu|facer|acer_wmi'

```

Do not load `facer` and `linuwu_sense` at the same time.

## Rollback

Uninstall the patched module:

```bash
sudo make uninstall

```

Or load back the default kernel module:

```bash
sudo rmmod linuwu_sense 2>/dev/null || true
sudo modprobe acer_wmi

```

## Notes

This patch focuses specifically on the Acer Nitro AN515-58. Other models may use different WMI payloads, so do not assume it is safe for all Acer Nitro/Predator laptops without testing.

References used for the RGB pipeline:

* JafarAkhondali `acer-predator-turbo-and-rgb-keyboard-linux-module`
* `docs/facer_rgb_reference.py`
