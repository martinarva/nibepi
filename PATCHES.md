# Patches on top of `anerdins/nibepi#1.2.1`

Branch `arva`. Every change is tagged in the source with a `NIBEPI_PATCHED*`
comment so it can be found again later.

Hardware: Novelan L12 SPLIT / Nibe SHK200S, firmware 9696, F-series protocol.

Note that `1.2.1` upstream is a **branch, not a tag**.

## backend.js

**1. The `close` event never reached Node-RED.** A USB glitch fires `close` on
the serial port, not `error`, and the existing handler only logged it. No fault
was ever reported, so the parent's restart path never ran and the core sat on a
dead port until the next reboot. Both `close` and `error` now report the fault
once and exit, which triggers the vendor's own `handleCore(config, true)`.
Measured: recovers on its own in about 12 seconds.

**2. serialport 13 API.** v9 exported the class directly and took
`new serialport(port, baud)`; v13 exports `{ SerialPort }` and takes an options
object. v13 also ships prebuilt binaries, so there is no native compile step —
v9 would not build against Node 16 or 14 at all, since `v8::Object::Set` was
removed.

## index.js

**3. MQTT re-subscription.** The client uses `clean: true`, but subscriptions
were only set up inside a promise `.then()`, which resolves exactly once. Every
broker reconnect therefore lost all subscriptions silently — room temperature
simply stopped arriving until Node-RED was restarted. Subscriptions are now
re-established on every `connect` event.

**4. Clearing alarm 251 requires an edge.** The pump clears "Kom.avb Modbus"
only on a **0 → 1 transition** of register 45171. nibepi always wrote 1, so only
the first reset after a reboot ever worked. It now writes 0, then 1 two seconds
later.

If the alarm ever sticks again, publishing `0` to `nibe/modbus/45171/set`
by hand makes nibepi's next write of 1 a real edge.

**5. `resetCore()` did not clear `regQueue`.** The poll list is parent state and
survived a core restart, so `addRegular()` considered every register already
added and never sent `regRegister` to the replacement core. After any restart
only about 16 of the 87 configured registers were still polled; the rest went
stale in Home Assistant without any error. `resetCore()` now clears it.

**6. MQTT auto-discovery rewritten** (`NIBEPI_PATCHED_DISCOVERY`). The original
mapped only 5 units and gave `A` a `device_class: power`, which Home Assistant
rejected outright (`unit A is not valid together with device class power`) —
dropping the entity entirely, with a single log line as the only symptom. 13
units now carry a valid `device_class` + `state_class` pair, plus `unique_id`
(`nibepi_<register>`), `object_id`, and a `device` block so everything lands
under one device instead of scattering.

> **`index.js` uses CRLF line endings.** Patch scripts must open it with
> `newline=""` and join with `\r\n`, otherwise exact string matching fails
> silently and the patch appears to succeed while changing nothing.

Watch the unit strings: `l/m` is not accepted for `volume_flow_rate` — Home
Assistant wants `L/min`, and rejects the whole entity if you get it wrong.

## models/SHK200S.json

**7. 346 broken degree signs.** This one model file had a literal U+FFFD
(`EF BF BD`) where `°` belonged, so every temperature arrived in Home Assistant
as `�C` and InfluxDB recorded them under a separate bogus measurement. The
other model files are clean UTF-8. Verified that every occurrence was followed
by `C` before replacing with `C2 B0`.

## package.json

**8.** `serialport` bumped to `^13.0.0`; `modbus-serial` removed. The latter was
the only reason the old native toolchain was pulled in at all — it is used only
by the dead `modbus-*.js` files, which `index.js` never loads.

## index.js — startup reads

**9. The startup read burst** (`NIBEPI_PATCHED_STARTUP_READS`). Whenever the poll
list is (re)built — at startup and after every core restart — `addRegular()` runs
for all ~87 configured registers at once, and it used to fire an immediate read for
each. The pump answers one register at a time (about 1.4 s each), so the queue took
some two minutes to drain and the reads at the back timed out after 37 s: a burst of
~37 `No respond from register` lines about 36 s after every start. One of the
casualties was the single `supply_s1` read that every plugin in
`node-red-contrib-nibepi` registers on (see its PATCHES.md), which made plugin
registration after a restart a matter of luck.

Those immediate reads were redundant — the regular poll reads the same registers
within one cycle anyway. Bulk (re)adds now skip them (`addRegular(address, false)`);
a single register added from the UI still reads at once.

Measured after a restart: 0 timeouts in the first 75 s (was ~37), and RMU, indoor
and forecast control all registered within 75 s without the watchdog having to act.

## index.js — persistence under Docker

**10. Settings and graphs were never written to disk in a container**
(`NIBEPI_PATCHED_PERSIST`). `updateConfig()` only writes directly when `docker`
is true; otherwise it first runs `sudo mount -o remount,rw /` and skips the write
if that fails. Nothing calls `setDocker()` in a plain Node-RED image, so `docker`
stayed false, and a container has no `sudo`: every save failed silently. Settings
changed in the UI lived in memory only and were gone after the next restart —
`config.json` had not changed once in five weeks. `saveGraph()` had no container
branch at all, so graphs were never saved either and every restart emptied every
chart.

`docker` is now detected from `/.dockerenv`, and under Docker graphs are written
directly, through a temp file and a rename so a restart mid-write cannot leave a
truncated `graph.json`. Enable "Save graphs" (`system.save_graph`) to use it.

Verified: a setting changed through the same path the UI uses reached
`config.json` within the 5 s debounce, and after a restart the forecast chart came
back from `graph.json` before the first new forecast run.
