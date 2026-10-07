# rtw88 monitor-mode fixes: channel and control-frame filter lost on interface restart

Two small fixes for the in-kernel `rtw88` driver. Both share one root cause: after an
interface stop/start cycle the chip is reset by `power_off()` / `power_on()`, but **mac80211
only calls `config()` and `configure_filter()` when its own state changes**. After the cycle
nothing has changed from mac80211's point of view, so neither the radio channel nor the
control-frame filter is reprogrammed.

The user-visible effect is that `iw` reports the configured channel while the card listens
elsewhere, with no error anywhere.

| defect | symptom | patch |
|---|---|---|
| channel not reprogrammed | 3 frames instead of thousands | 0001 |
| control-frame filter not restored | 0 RTS, 0 CTS, 0 ACK | 0002 |

Patch 0002 only applies on top of commit `ed51a86b787f` ("wifi: rtw88: Enable receiving
control frames in monitor mode", v7.3-rc1), which enables the filter but does not restore it
after the chip is powered back on.

## Submitted upstream

Sent to `linux-wireless` on 2026-10-07. Thread:
<https://lore.kernel.org/linux-wireless/20261007091225.413-1-jeremy.fareau@gmail.com/>

The `.patch` files here are byte-identical to what was sent.

## Test setup

Raspberry Pi 4B, aarch64, Ubuntu 26.04.1, kernel `7.0.0-1020-raspi`, in-kernel `rtw88` with
`ed51a86b787f` backported. Two adapters on the same host, both in monitor mode on channel 6,
captured in parallel:

* ALFA AWUS1900 — RTL8814AU, `rtw88_8814au`
* Linksys WUSB6300 — RTL8812AU, `rtw88_8812au`

## Isolating defect 1

Six runs, full USB unbind/rebind between each, 25 s captures:

| run | sequence after `ip link up` | frames |
|---|---|---|
| 1 | `set channel 6` | 3 |
| 2 | `set channel 6` again | 0 |
| 3 | `set channel 6` twice in a row | 3 |
| 4 | no `set channel` at all | 3 |
| 5 | `set channel 11` — a real change | **431** |
| 6 | **WUSB6300 control**, same sequence as run 1 | **2438** |

Only an actual channel *change* reaches the hardware. Setting the channel mac80211 already
believes is current is a no-op, so the radio stays wherever `power_on()` left it.

## Both patches applied

Five consecutive `down`/`up` cycles, 25 s captures:

| cycle | AWUS1900 total · control | WUSB6300 total · control |
|---|---|---|
| 1 | 6639 · 4556 | 2274 · 1506 |
| 2 | 9622 · 7754 | 2937 · 2214 |
| 3 | 6847 · 4985 | 2166 · 1475 |
| 4 | 7420 · 5542 | 2493 · 1789 |
| 5 | 7703 · 5250 | 2440 · 1625 |

Without the patches the control columns are **0** on both adapters, and the AWUS1900 total
drops to 0–4 from the first cycle on.

## What was verified

* `rtw_set_channel()` takes no lock and every caller already holds `rtwdev->mutex`, so the
  added call in `rtw_ops_start()` is correctly serialised.
* The idiom is pre-existing: `rtw_ips_pwr_up()` already calls `rtw_core_start()` followed by
  `rtw_set_channel()`. The IPS resume path was covered; the `ieee80211_ops::start()` path was
  not. The channel fix was deliberately placed in `rtw_ops_start()` rather than in
  `rtw_core_start()` so that the IPS path, and its `rtw_coex_ips_notify(COEX_IPS_LEAVE)`
  ordering, is left untouched.
* Station mode is unaffected: scans return 28 then 34 BSS (AWUS1900), 10 then 11 (WUSB6300).
* Three `rmmod` / `insmod` cycles, clean.
* `dmesg`: no WARNING, BUG, Oops or call trace.
* `checkpatch.pl --strict`: 0 errors, 0 warnings, 0 checks on both patches.
* Build: no code warnings.

## What was NOT verified

* **PCI and SDIO paths.** Patch 0002 touches `rtw_core_start()`, shared by all bus types.
  Only USB was exercised — no PCI or SDIO hardware was available.
* **lockdep.** `CONFIG_PROVE_LOCKING` is not enabled on the test kernel. The absence of a
  deadlock rests on code review and on use, not on instrumentation.
* **Other chips.** Only RTL8814AU and RTL8812AU were exercised. Patch 0002 also concerns
  RTL8723D, RTL8703B and RTL8821A by construction.
* **Bisection.** The `Fixes:` tag on patch 0001 was derived by reading the history, not by
  bisecting.

## Tooling disclosure

Per `Documentation/process/generated-content.rst` and
`Documentation/process/coding-assistants.rst`: the diagnosis, the code and the commit
messages were produced by an AI coding assistant in an interactive session. All measurements
were taken on real hardware. Both patches carry `Assisted-by: LLM`; the `Signed-off-by`
tags were added by hand, because only a human can certify the DCO.

## Applying

```
git checkout -b rtw88-fixes master      # master at 69f80fef3153 (v7.3-rc7)
git am 0001-*.patch 0002-*.patch
```
