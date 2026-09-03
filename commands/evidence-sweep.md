---
description: Capture all available evidence from the device before changing anything (phase 1 of linux-phone-porting)
argument-hint: "[symptom being investigated]"
---

Run phase 1 of the `linux-phone-porting` skill against the device for: $ARGUMENTS

Do not propose a fix in this command. The output is evidence, not a diagnosis.

1. **Before reproducing**, enable the debug knobs relevant to the suspected subsystem (`debug_mask`, dynamic debug, `drm.debug`, tracepoints). A second reproduction costs another boot, so turn everything on now.
2. Capture the full dmesg of the failing run to a file on the host.
3. If the device booted far enough for systemd, harvest `/var/lib/systemd/pstore/` — **not** `/sys/fs/pstore/`, which `systemd-pstore.service` empties early. Copy any console record to the host immediately; the slot is usually singular.
4. If the device wedged, note whether a kernel-level console (`CONFIG_U_SERIAL_CONSOLE` + `console=` on the cmdline) was actually active. A userspace getty enumerating over USB is not evidence that a console existed.
5. Read back what is actually flashed and hash the payload. Do not take the build log's word for it, and do not compare image sizes — padding makes them equal.
6. For each log line you intend to rely on, ask whether your own observation produced it. Note anything sampled from a crash-state or debug node while the subsystem was live.
7. When a firmware interface rejects a call, capture the raw response words — dynamic debug or a trace in the call wrapper — not just the mapped errno. The mapping cannot distinguish a wrong encoding from a policy refusal from a not-found, and those have opposite next steps.
8. When a benchmark or score is part of the capture, record its control condition: what else could hold the resource (a compositor holding DRM master, another client on the device), and whether it held for every run. A number produced under an uncontrolled condition is void.

Report: what was captured, where it is on the host, which channels were unavailable and why, and which lines are suspect under step 6.
