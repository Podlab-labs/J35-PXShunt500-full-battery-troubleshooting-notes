# J35 / PXShunt500 not reporting full battery — troubleshooting notes

**Date:** 2026-09-20 (updated same day — tail current change applied)

## Setup
- 2023 Jayco Penguin
- BMPRO J35 power management system (has its own built-in PWM solar controller, not in use)
- Victron SmartSolar MPPT 100/50 doing the solar charging instead
- BMPRO PXShunt500 external shunt + PX Gateway-II CANbus, installed so the J35 still reports battery state despite charging coming from the Victron rather than the J35's own PWM controller
- Battery: LiFePO4, 100Ah

## Symptom
- Victron goes Bulk → Absorption (14.20V) → holds 2 hours (was fixed timer, tail current previously disabled) → Float (13.50V)
- Victron considers the battery full
- J35 never reports full, and its reported SoC decreases every day even though the Victron thinks the battery is topped up
- Loads on the van are all burst-type (fridge on gas, water pump, lights, TV, water heater) — nothing continuous
- No SoC/sync-related settings found in the J35 JHub app over Bluetooth

## Diagnosis — two candidate causes, in priority order

### 1. Wiring bypass (most likely, check first — still open)
The PXShunt500 manual is explicit: the shunt has a **battery-side** terminal (only the shunt's own flying lead goes here, straight to battery negative) and a **load-side** terminal, where "all loads, chargers and battery management systems" must land their negative.

If the Victron MPPT's negative output was wired directly to the battery negative post instead of to the shunt's load-side terminal, the shunt never sees any of the solar charging current — only the loads discharging the battery. That produces exactly this symptom: Victron is genuinely charging and correctly reports full, but the J35's coulomb counter only ever sees current going out, so it counts down every day regardless.

**Action:** Physically check (or have the installer check) where the Victron's negative wire is landed — should be on the shunt's load-side terminal bus, not directly on the battery negative post. Same check for the DC-DC charger or any other charge source if present.

### 2. Coulomb-counter resync/full-detect timing
The PXShunt500 documentation confirms it uses an internal "learning algorithm" over charge/discharge cycles rather than an exposed tail-current/full-voltage setting (unlike a Victron SmartShunt/BMV). There's no user setting found for this on the J35 JHub app or apparently the panel.

With tail current previously disabled and absorption ending on a fixed 2-hour timer regardless of current, the shunt may never have seen "voltage near peak + current near zero" simultaneously — especially with burst loads potentially interrupting a low-current window right when it matters. This would mean the shunt's full-charge resync never fires, so its SoC counter keeps drifting from an increasingly wrong baseline.

**Change applied (2026-09-20):** Set Victron tail current to **3.0A** (actual amps, ~3% of the 100Ah capacity — not a percentage field, not relative to the 100A/500A device ratings). This should let absorption end early (before the 2-hour max) once real charge current tapers to ≤3A, giving the shunt a genuine full/low-current moment to detect.

**Absorption Time mode — kept on Fixed, not switched to Adaptive.** Victron's "Adaptive" absorption algorithm is a lead-acid-specific feature (scales duration from idle battery voltage / bulk-phase length — sulfation/gassing management for flooded/AGM/gel). It isn't designed or documented for lithium, and LiFePO4's flat voltage curve makes those lead-acid heuristics meaningless here — using it could produce erratic absorption durations unrelated to actual charge current. Victron's own factory default for lithium is Fixed 2-hour absorption; they normally pair it with tail current = 0A specifically to *guarantee* the full 2 hours for cell balancing. So the trade-off of our tail-current change: absorption may now end earlier on days current tapers quickly, giving less full-duration balance time. Worth periodically checking individual cell voltages (if the BMS/app exposes them) to make sure they're not spreading apart over time; if they do, occasionally let one charge cycle run the full fixed 2 hours (temporarily raise/disable tail current) to force a complete balance session.

**Remaining action if wiring checks out and tail current alone doesn't fix it:**
- Contact BMPRO support (03 9763 0962) and ask directly whether the PXShunt500/J35 has a manual "sync to full" or reset function — not documented publicly, but many similar systems have one.

## Current Victron settings (as of 2026-09-20)
- Absorption voltage: 14.20V
- Absorption time: **Fixed**, 2 hours max
- Tail current: **3.0A** (changed from disabled)
- Float voltage: 13.50V

## Sources consulted
- PXShunt500 Owner's Manual (BMPro): https://teambmpro.com/wp-content/uploads/pxshunt500-manual.pdf
- How do I connect the XShunt500 and the PXGateway? (BMPRO Zendesk): https://bmpro.zendesk.com/hc/en-us/articles/11669038548879-How-do-I-connect-the-XShunt500-and-the-PXGateway
- BMPRO Power Management Systems FAQ: https://teambmpro.com/battery-management-systems-faq/
- Victron Community forum — "Please fix my settings, 100% sync not happening": https://community.victronenergy.com/t/please-fix-my-settings-100-sync-not-happening/33550?page=2
- Victron "Adaptive Charging: how it works" whitepaper: https://www.victronenergy.com/upload/documents/Adaptive_charging_how_it_works.pdf
- Victron BlueSolar/SmartSolar 100/30, 100/50 manual — Operation section (lithium defaults): https://www.victronenergy.com/media/pg/Manual_BlueSolar_100-30__100-50/en/operation.html
- Victron Community archive — "Absorption time and LiFePO4, what is the best": https://communityarchive.victronenergy.com/questions/96257/absorption-time-and-lifepo4-what-is-the-best.html

## Next steps (open)
- [ ] Confirm Victron negative wiring destination (shunt load-side vs. battery post direct)
- [ ] Monitor J35 SoC over the next several charge cycles now that tail current is 3.0A — does it reach 100%?
- [ ] Occasionally check individual cell voltages for balance drift, given tail current can now end absorption early
- [ ] If still not syncing after wiring check + tail current trial, call BMPRO support re: manual sync/reset option