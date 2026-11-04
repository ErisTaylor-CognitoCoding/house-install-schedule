# How the house works

Read this before ANY change or suggestion on the house. If a change alters how something works, update this file in the same pass.
Sources: `index.html` (install guide), `next-week.html` (half-term page), memory files, and live config read 4 Nov 2026. Where memory and live config disagreed, live config is written here and the conflict is noted.

---

## 1. What runs where

**Elysium** — hostname `elysium`, LAN `192.168.1.146`, Tailscale `100.100.73.45`. Runs everything below in Docker.

| Container | Job | Notes |
|---|---|---|
| `homeassistant` | Home Assistant Container, `:stable`, host network | Config `~/projects/homeassistant/config`. No add-on store. Never propose HA Supervised / HAOS VM / VT-x — rejected twice by Zero (13–14 Aug 2026). |
| `zigbee2mqtt` | Zigbee, Sonoff ZBDongle-P (zstack), channel 11, transmit_power 20 | `~/projects/zigbee2mqtt/`. Panel on host port 8099. |
| `mosquitto` | MQTT broker, `192.168.1.146:1883` | User `nova`; password in Z2M `configuration.yaml`. |
| `esphome` | Voice PE firmware | Configs: `home-assistant-voice-09c7c5.yaml`, `-bedroom.yaml`, `-office.yaml`. |
| `matter-server` | Matter, host network | Nothing paired to it. |
| `whisper` | STT, `small-int8`, beam 2 | `base-int8` dropped whole sentences (12 Sep 2026). |
| `piper` | Local TTS, `en_GB-alan-medium` | Nova's own voice goes through Nova's `/api/tts`, not piper. |
| `ps5-mqtt` | PS5 rest/wake | `~/projects/ps5-mqtt/`, local playactor patch. |
| `pihole` | DNS for Elysium, cosmos-hq, cosmos laptop ONLY | Not the LAN's DNS, on purpose. |
| `nginx-proxy` | `*.cognitocoding.com` hostnames | Only resolve via Pi-hole. |
| `nova` | Nova; holds `HA_TOKEN` | Nova has `ha_*` MCP tools. |

**Network (4 Nov 2026)**
- No broadband yet. Router is a TP-Link Archer MR600 (4G) with a Poynting XPOL-2-5 antenna, gateway `192.168.1.1`, SSID `EE-MX82JZ`. It replaced the EE 4G hub with the same SSID and password.
- TP-Link RE505X extender added downstairs 3–4 Nov 2026, same SSID.
- MR600 2.4 GHz is pinned to channel 11 / 20 MHz, auto-channel off (3 Nov 2026). Channel 1 sat on top of Zigbee channel 11 and broke the Bedroom 3 valve. Set the RE505X and the future broadband router the same way.
- 21 DHCP reservations on the MR600 (3 Nov 2026). Admin login is through the `tplinkrouterc6u` lib, username `admin`.
- Elysium has no Wi-Fi of its own. It is online through an Alfa AWUS036AXM USB dongle (`mt7921u`, interface `wlx00c0cab9c757`). The dongle MUST sit in a black USB 2 rear port; on USB 3 it never comes up. Live: `eno1` has no carrier; the default route is the dongle; the dongle holds `.146`. (Memory said Wi-Fi would get `.149`. Live config says `.146`.)
- Plan: Elysium goes on a wired data point at the Garage server desk. Physical location today: unverified.
- `/usr/local/sbin/wifi-watchdog`, run every minute by a timer:
  - No internet for 3 min → USB re-plug of the dongle.
  - Kernel logs `mt7921u ... failed:-110` (driver wedged) → forced reboot, NEVER a re-plug. A re-plug of a wedged dongle froze Elysium at 15:58 on 4 Nov 2026.
  - A re-plug that hangs for 60 s → forced reboot.
  - Hub unreachable after a re-plug → reboot, at most once per 30 min.
  - 4G outage (hub answers, no internet) → re-plug every 15 min, never reboot.
  - Alerts go to ntfy topic `elysium-ht5nauq27ayyfma`.

**If Elysium is down, these still work**
- Edge-mode lights from the wall: Kitchen 1 and 2, stairs bottom and top, Garage.
- Bedroom 2 and 3 wall switches (plain power), and the fans' own remotes.
- Smoke and CO alarms. They interlink by radio and need no server.
- Heater frost stats (Garage, summer house).
- Everything in the Tuya app.
- The Ring app and its cloud.

**If Elysium is down, these do not work**
- The Lounge and Master Bedroom wall switches. A Detach press goes Z2M → HA → tuya_local, so the rocker does nothing. Fallback is the Tuya app. Zero accepted this, 4 Nov 2026.
- Every dashboard and tablet.
- Every automation, including the smoke alert to phone and tablets.
- Voice / "Hey Nova".
- Radiator schedules and external-sensor feeds. The valves keep their last target.

---

## 2. Rooms

HA area names: Living Room (alias Lounge), Kitchen, Master Bedroom, Bedroom 2 (Salia's room), Bedroom 3 (aliases Spare Room, Ayub's Room), Hall, Garage, Summer House, Office, Front Door, Front Garden, Back Garden, Bathroom. Two more areas sit on a separate "28 Courageous" floor: 28 Courageous and Cat Shed. That floor is the old house; Marshmellow lives there.

### Kitchen
| | |
|---|---|
| Fixtures | Two plain 33 W 6000 K panels, one per rose. One 2-gang BSEED rocker. |
| Modules | `Kitchen 1`, `Kitchen 2` (ZBMINI R2) in the roses. **Edge.** Wrapped by switch_as_x as `light.kitchen_1` / `light.kitchen_2`. |
| Why | One switch drives two lights on separate cables, and there is no neutral at the switch. So there is one module per rose and one rocker per module (6 Oct 2026). One rocker cannot drive two modules. |
| Sensors | Kitchen Presence (06P), Kitchen Temperature (02B), Kitchen Cat Flap (DJT11LM), Washing Machine (DJT11LM), Fridge and Freezer (02LD probes; only the probe goes inside), Back Door Sensor (04PR2). |
| Heat | Kitchen Radiator (TRVZB): 17 °C 06:00, 14 °C 22:00. Drops to 5 °C while the back door is open. |
| Other | Sox's litter box (Tuya). PetSafe microchip flap in the back door. Voice PE `home_assistant_voice_09c7c5` (`.246`), live on Hey Nova. |
| Screen | Apolosign 21.5" planner `.40` → `kitchen-dash`. |

### Living Room / Lounge (the office is a desk in here)
| | |
|---|---|
| Fixtures | ycwdcz Tuya fan + light, via tuya_local at `.213` (profile `blitzwill_ceilingfanlight`). Two 1-gang 2-way rockers (two doors). |
| Module | `Lounge Fan Module` (ZBMINI R2). **Detach**, power-on behaviour `on`. |
| Why | Cutting power drops the smart fan off Wi-Fi. Detach keeps mains on the fan, and the rocker only sends `toggle`. NEVER switch it to Edge. |
| Sensors | Lounge Presence (06P, sensitivity high, timeout 300 s), Lounge Temperature (02B). |
| Heat | Living Room Radiator: 18 °C at 10:00, 16 °C at 22:00 ("dog only"). Zero turns it up on the tablet. |
| Tablets | Lounge `.163` → `lounge-dash`, on `Lounge Tablet Socket` (MINI-ZB1GS) for 80/40 charging. Office `.190` → `office-dash`, on Zero's desk. |
| TVs | Philips 55OLED760 `media_player.lounge_tv` / `remote.lounge_tv` (`.191`) plus Lounge Fire Stick (`.97`). Hisense 43" "PlayStation TV" (`.65`) plus PS5. TV sockets are permanent live, never on a module. |
| Voice PE | The 4th unit is for this room. Not in HA as of 4 Nov 2026 (unverified whether it is set up). |

### Master Bedroom
| | |
|---|---|
| Fixtures | CHANFOK 16" fan + light, tuya_local `.106` (Yunlong profile; the Chanfok profile wants DP 53, which this unit never sends). 1-gang 1-way rocker. |
| Module | `Master Bedroom Fan Module`. **Detach**, power-on `on`. Same reason as the Lounge. |
| Sensors | Master Bedroom Presence (06P), Master Bedroom Temperature (02B). |
| Heat | 20 °C at 06:00, 14 °C at 09:00 — he sleeps cold. |
| Tablet | `.247` → `bedroom-dash`. Charges from the bed's USB; no socket, no module. |
| TV | Bedroom Fire Stick 4K (`.250`). Install page puts the JVC Fire TV Edition here. HA still lists the JVC built-in as `media_player.lounge_fire_tv` (`.221`, unavailable) in Living Room; unverified. `media_player.bedroom_eris_tv_bedroom_tv` is unavailable. |
| Voice PE | `bedroom_voice` (`.153`). Unavailable at 17:34 on 4 Nov 2026. |

### Bedroom 2 (Salia's room — Salia is he/him)
| | |
|---|---|
| Fixtures | CHANFOK fan + light, tuya_local; options host `.110` overrides data host `.116`. |
| Module | `Bedroom 2 Fan Module`. **Plain power — detach off (4 Nov 2026, Zero's call).** The wall switch is plain power. The boys use the fan's own remote. |
| Why | Not in `fan_modules_always_on` or `wall_switch_fan_lights`. Never add it back. |
| Sensors | Bedroom 2 Presence (06P), Bedroom 2 Temperature (02B). |
| Heat | No valve: the bed is in the way (3 Nov 2026). |
| Tablet | Fire HD 8 on order. `bedroom-2-dash` exists. |

### Bedroom 3
| | |
|---|---|
| Fixtures | CHANFOK fan + light, Tuya **cloud** only: `light.bedroom_3_light`, `light.bedroom_3_night_light`. No `fan.` entity in HA. |
| Module | `Bedroom 3 Fan Module`. **Plain power — detach off (4 Nov 2026).** Same as Bedroom 2. |
| Sensors | Bedroom 3 Temperature (02B). Presence still to place. |
| Heat | Bedroom 3 Radiator at 5 °C "for now", on the room sensor. |
| Tablet | On order. `bedroom-3-dash` exists. |

### Hall and stairs
| | |
|---|---|
| Fixtures | Two plain LED lights. Bottom plate is 2-gang 2-way; top plate is 1-gang 2-way. |
| Modules | `Stairs Bottom Light`, `Stairs Top Light`. **Edge.** switch_as_x lights. |
| Why | One module per light, so the house can switch the bottom one alone (the doorbell-after-dark automation). No Hue anywhere since 8 Oct 2026. |
| Sensors | Front Door (04PR2), Hall Temperature (02B). Hall and landing presence still to place. |
| Safety | Ei3016 smoke alarms in the hall and on the landing. Ei3018 CO alarm downstairs by the boiler. All on radio interlink. Ei428 relay → `Smoke Alarm Relay` ZBMINI (detach on, reports only), area Hall. |
| Heat | Hall Radiator at 18 °C. Drops to 5 °C while the front door is open. |
| Tablet | `.122` → `hall-dash`. Fully Plus licensed 3 Nov 2026. Locked upside-down (power lead at the bottom, Zero's choice, 4 Nov 2026): `forceScreenOrientation=3` (Reverse Portrait, value read off Fully's own settings page) + `forceScreenOrientationGlobal=true` so the Ring app follows. |

### Garage
| | |
|---|---|
| Fixtures | CHANFOK fan/light (29 Oct 2026), on module `Garage Light`, detach **off** (Edge). The fan is not in the Tuya app yet. It came in via the Chanfok Smart app and has to be re-paired in Tuya. |
| Conflict | The rule says a smart fitting → Detach. The install page says "if a smart fitting goes in it becomes Detach". Live is Edge. Unresolved: do not change it without Zero. |
| Sensors | Garage Temperature (02WD sealed), Garage Back Door Sensor (04PR2), Garage Door up-and-over (04P). Presence still to place. |
| Heat | Devola wall heater with its own frost stat, on a permanently live fused spur (Always On, no module). `climate.garage_heater` is unavailable. |
| Camera | Ring Stick Up Cam, `camera.garage_live_view`. |
| Tablet | To buy (server desk). |

### Summer house (stays a summer house — 27 Oct 2026)
| | |
|---|---|
| Fixtures | No lighting circuit and no module. Two FULEN battery motion lights (to buy), not in HA. No fan: the roof is too low. |
| Heat | Devola 2.5 kW on a 13 A plug into a socket (no spur). Its own thermostat holds 5 °C with the server down. |
| Sensors | Summer House Temperature (02WD), Summer House Door Sensor. |
| Other | Solar Ring camera to buy. Fire HD 8 on order. Air con: only a 5,000 BTU unit is ever permitted (4.46 m² room). |

### Outside and other
- Back Gate 04P in a weatherproof case. It reaches the network via the Garage module. Check the seal when the battery is changed.
- Ring at the house: Front Door doorbell (`_2` entities, Doorbell 3 Plus), Backdoor, Path, Garden, Lounge Pet Cam, and `camera.office_live_view` (named "Summer House Live view"). Unverified which physical camera that last one is: the install page lists the summer-house Ring as "to buy".
- 28 Courageous (old house): 28 Front Door (`camera.front_door_live_view`, no `_2`), 28 Spotlight, Marshmallow cam.

---

## 3. Rules and decisions

| Rule | Why | Decided |
|---|---|---|
| Module mode follows one question: would cutting power break the fitting? No → **Edge**. Yes, with a wall switch → **Detach**. Yes, no wall switch → **Always On** (no module). | Edge kills a smart fan's Wi-Fi. Detach on a dumb light leaves it uncontrollable from the wall when the server is down. | Install page |
| Lounge and Master Bedroom fan modules stay **Detach** and stay ON (`fan_modules_always_on`). | Detach keeps mains on the smart fan. Edge would cut its power. | 9 Oct 2026 |
| Bedroom 2 and 3 fan modules are **plain power**, not Detach. Never add them to the Detach automations. | Zero: the boys use the fans' remotes. The wall switch is plain power. | 4 Nov 2026 |
| Leave modules on factory settings at the wall. Edge/Detach is set in software from HA/Z2M. | Nothing to do on site. | Install page |
| Never neutral or earth on S1/S2. No touch switches on S1/S2. The module goes wherever the neutral is (rose if the switch box has none). | Damages the module; a signal pair can't power a touch switch. | Install page |
| TV sockets (Lounge Philips + stick, Hisense + PS5, bedroom) are permanent live, never on a module. | The stick and the PS5 must stay on the network to turn the TV on. | Install page |
| Tablet sockets cut at 80 % and back on at 40 %. | A tablet held at 100 % swells its battery. | Aug 2026 |
| Smoke/CO alarms never depend on HA. HA only adds a phone/tablet warning on top. | Safety layer works with power, server and network down. | Install page |
| No camera on any tablet: `disableCamera=true`, camera motion detection off, HA `camera.<tablet>` disabled. Wake by touch or a Zigbee presence sensor. | Zero: "creepy". Tablets go in the children's bedrooms. Never re-enable. | 29 Oct 2026 |
| Say **Tuya app**, never Smart Life. | Zero has no Smart Life. | 29 Oct 2026 |
| Links handed to Zero on his PC: `https://homeassistant.cognitocoding.com`. Fire tablets and other LAN devices: `http://192.168.1.146:8123`. Away from home: `http://100.100.73.45:8123`. | Only Elysium, cosmos-hq and the cosmos laptop use Pi-hole. Never suggest changing the router's DHCP DNS. | 19 Aug 2026 |
| HA `internal_url` stays `http://192.168.1.146:8123`. | Voice PE fetches TTS from it. The hostname silently killed bedroom speech. | 19 Aug 2026 |
| It is a lived-in house: no "while the floors are up", no first-fix cable runs. Prefer wireless or plug-in. | No floors are up. Zero has corrected this more than once. | 9 Oct 2026 |
| Say "the house" / home. Never "the new house" or "after the move". | He lives there. | 7 Oct 2026 |
| Summer house stays a summer house: plug-in heater, battery lights, no fan, no module. | Zero's call. | 27 Oct 2026 |
| There is no Office room. The office is Zero's desk in the Lounge. HA keeps an "Office" area on the Downstairs floor for `office-dash` and the Office tablet. | Zero's call. | 29 Oct 2026 |
| No Hue anywhere. No `emulated_hue`. | Zero rejects anything Hue-named. Alexa reaches Tuya through the Tuya app's Alexa skill. | 8 Oct and 25 Oct 2026 |
| Garage up-and-over uses a contact sensor, not a tilt sensor. Gate and up-and-over use the older 04P, not the PR2. | Tilt reports seconds late. The PR2 misses the first action after hours idle. | Install page |
| Presence is radar (06P), mains USB-C, within a lead of a socket. | PIR loses people sitting still. | Install page |
| Fans use two temperatures (on above, off below). | One figure makes the fan chatter on and off. | Aug 2026 |
| Bedroom presence lights are off-only; never switch on by themselves. | A self-switching light gets its sensor unplugged in week one. | Aug 2026 |
| Living Room dusk lamp triggers only on the way down. Off comes from the room emptying. One go per sitting. | Otherwise it lights the room and turns itself off. | Aug 2026 |
| The install page has ONE copy (this repo, GitHub Pages). No LAN IPs, `/dev/tty*` or `homeassistant.cognitocoding.com` on it. Update its Done list whenever something pairs. | Tony reads it, and a stale duplicate nearly overwrote it. | 18 Aug 2026 |
| Never relay a pairing PIN (VIDAA etc.) through chat. Zero types it in the HA UI. | PIN lasts 15 s. | 24 Oct 2026 |
| Never turn ADB off on the Fire Sticks or drop HA from them. | Zero bought them for HA control. | 24 Oct 2026 |
| Voice PE configs: never add `api: encryption:`. | Bricks the HA connection. | 17 Aug 2026 |
| Cat names: **Marshmellow** in HA (Tractive spells it Marshmallow; leave that). Sox lives at the house (Kitchen flap). Marshmellow stays at 28 Courageous. | — | 26 Sep 2026 |
| Lounge dashboard shows only the lounge light and fan, never other rooms' lights. | Zero's request. | 9 Oct 2026 |
| Apolosign: leave Fully `kioskMode` OFF unless Zero says otherwise. | Kiosk lock knocked it off Wi-Fi and locked him out of settings. | 6 Oct 2026 |
| Boiler: Potterton combi. Hive SLR1c plan dropped; an Optimum OP-WFSTAT kit was ordered. | Old stat was not the Wi-Fi version. | 2 Nov 2026 |

---

## 4. Automations (`automations.yaml`, 4 Nov 2026)

| id | What it does | Why |
|---|---|---|
| `front_door_motion_show_cameras` | 28 front door motion → Apolosign loads `kitchen-dash/cameras-28?kiosk` for 1 min, then back. | Watch the old house from the kitchen. |
| `stairs_light_doorbell_after_dark` | Doorbell after sunset → bottom stairs light on 3 min, unless it was already on. Also fires on the polled `sensor.front_door_last_activity_2` (ding < 5 min old). | Ring push can fail. The poll is a 1–2 min fallback. |
| `lounge_fan_warm_occupied` | Blueprint: Lounge fan at 50 % when occupied and > 22 °C. Stops < 20.5 °C or 5 min empty. | Two thresholds stop chatter. |
| `lounge_empty_tv_light_off` | Lounge empty 10 min (+5 min sensor hold) → fan light off; if the stick is awake, TV Standby + stick sleep. Skips while the stick plays media (ADB `dumpsys audio`). Rechecks every 5 min. | The radar lost Zero sitting still and killed the TV mid-watch (10 Oct 2026). The androidtv state reads `idle` during playback, so never use it. |
| `wall_switch_fan_lights` | MQTT `toggle` from `Lounge Fan Module` / `Master Bedroom Fan Module` → toggles that fan's light. | Detach rockers only report a press. Bedroom 2/3 removed 4 Nov 2026. |
| `lounge_tablet_charge_limit` | Blueprint: `switch.lounge_tablet_socket` off at 80 %, on at 40 %. | Battery health. |
| `bedrooms_empty_light_off` | Bedroom 2 and Master Bedroom light off after 15 min empty, including after HA start (power cut). | Off-only rule. A power cut turns fans/lights on (power-on `on`). |
| `upstairs_hall_light_sun` | `light.stairs_top_light` on 30 min before sunset, off 30 min after sunrise. | Stands in for the planned presence lights (sensors not placed). |
| `playstation_tv_follows_ps5` | PS5 on → Hisense on via `vidaa_tv`, then HDMI2 (retries once). | CEC is dead on this Hisense. |
| `playstation_tv_off_with_ps5` | PS5 on → off → TV off, only if the TV is on HDMI2. | Don't kill the TV when it's on something else. |
| `smoke_co_alarm_phone_alert` | `Smoke Alarm Relay` toggle → `input_boolean.smoke_alarm_sounding` (red banner on first views), urgent ntfy, HA app push, screen on + overlay + TTS on Lounge, Apolosign, Bedroom and Office tablets. Clears after 15 min. | Alarm reaches the phone. Tested with the real alarms 29 Oct 2026. The **Hall tablet `.122` is not in it** (open). |
| `fan_modules_always_on` | `switch.lounge_fan_module` or `switch.master_bedroom_fan_module` off for 2 s, or HA start → turn both back on. | A Detach module must never cut a smart fan's power. Bedroom 2/3 removed 4 Nov 2026. |
| `kitchen_radiator_back_door` / `hall_radiator_front_door` | Door open → valve to 5 °C (scene snapshot first). Closed → restore the snapshot. | Don't heat the street. Restore checks the scene exists (its state is `unknown`), fixed 3 Nov 2026. |
| `hall_ / kitchen_ / living_room_ / master_bedroom_ / bedroom_3_radiator_external_sensor` | Writes the room sensor into the valve's external temperature input on change, every 30 min and at HA start. | The valve heats to the room, not to the radiator. |
| `lounge_radiator_daytime` / `lounge_radiator_night` | 10:00 → 18 °C, 22:00 → 16 °C. | Zero's schedule. |
| `master_bedroom_radiator_schedule` | 06:00 → 20 °C, 09:00 → 14 °C. | He sleeps cold. |
| `kitchen_radiator_schedule` | 06:00 → 17 °C, 22:00 → 14 °C. | Zero's schedule. |
| `ring_push_watchdog` | Log "Shutting down push receiver" → reload the Ring entry after 2 min. | firebase_messaging gives up after 3 network errors. Needs `system_log: fire_event: true`, which is in config and loaded since HA started 16:22 on 4 Nov 2026. Not triggered yet. |
| `tablets_back_to_fully` | Every minute: any tablet whose foreground app has been something other than Fully (`de.ozerov.fully`) for 2 min gets its `Bring to foreground` button pressed. Covers all 5 tablets. | Zero, 4 Nov 2026: tablets must go back to the dashboard after Ring etc. Fully's own regain-focus timer didn't fire. Proven on the Lounge tablet 20:38 4 Nov 2026. |

**Blueprints** in `config/blueprints/automation/zero/`:
- `fan_when_warm_and_occupied`: used by the Lounge only.
- `tablet_charge_limit`: used by the Lounge only.
- `cat_flap_log`: not used. Watches the room inside a flap, because no flap can tell which way a cat went. Default timeout 15 min: a sleeping cat barely moves.
- The HA built-in `motion_light` was meant for the Garage, 600 s. Not used.

**On the install page but not built (4 Nov 2026):**
- Master Bedroom fan on temperature.
- Bedroom 2/3 fans on temperature.
- Garage presence light.
- Garage heater dew-point control.
- Cat flap log.
- Hall/landing presence lights.
- Living Room dusk lamp.
- Office tablet charge limit.
- Washing machine done.
- Fridge/freezer 30-min alerts.

**Scripts:** `fire_tv_open_app` (WAKEUP + HOME, wait, launch, re-launch if not focused), `pstv_with_vpn`, Netflix/iPlayer per room, `play_music` / `stop_music`.

---


## 5. Dashboards and tablets

| Tablet | IP | Start URL | Charging |
|---|---|---|---|
| Lounge ("Fire Tablet") | `.163` | `http://192.168.1.146:8123/lounge-dash/lounge` | Lounge Tablet Socket 80/40 |
| Master Bedroom | `.247` | `.../bedroom-dash/bedroom` | Bed USB |
| Office (Zero's desk, Lounge) | `.190` | `.../office-dash/office` | It drained on the PC's USB (29 Oct 2026). Half-term page says the desk double was fitted 29 Oct, yet it read 11 % while plugged in at 17:34 on 4 Nov 2026. Check what it is plugged into. |
| Hall | `.122` | `.../hall-dash/hall` | Unverified |
| Kitchen Apolosign 21.5" | `.40` | `.../kitchen-dash/kitchen?kiosk` | Mains |
| Bedroom 2, Bedroom 3, summer house | — | On order | — |
| Garage | — | To buy | — |

`wall-panel` (Main DashBoard), `tv-remotes` and `map` are not on any tablet's start URL.

**What each dashboard must and must not show (Zero's rules)**
- Rooms are separate **dashboards, not tabs** inside one. Built wrong once, 18 Aug 2026.
- `lounge-dash`: lounge light and fan only (never other rooms' lights), TV/stick, PlayStation section, thermostat. The pet cam was removed from the Lounge (18 Aug and 31 Oct 2026); do not put it back. The Garage cam is in its cameras subview.
- `office-dash`: title "Office", lounge light + fan, diary, front/back/garden cameras, lounge temp/humidity/presence, thermostat. **Never the TV or PS5.**
- `bedroom-2-dash` / `bedroom-3-dash`: **deleted 4 Nov 2026 (Zero: "scrap them").** No tablet pointed at them. Backups of the storage files are in `~/lovelace.bedroom_[23]_dash.bak-20261104`. Do not recreate them.
- `hall-dash`: doorbell camera, front door contact, doorbell battery, Hall + Top of Stairs lights, hall thermostat, "Before you go" (back door, back gate, garage door, garage back door, summer house, Sox, Marshmellow), hourly weather.
- `kitchen-dash`: family calendar, weather, date/time, shopping list, Kitchen 1/2, radiator, Front Door camera, Cats badge → `/kitchen-dash/cats` (map + per-cat tiles + "Kitchen flap last used"). Install and Half Term badges were removed 3 Nov 2026.
- Main views show the house door `camera.front_door_live_view_2`. The 28 door stays in each `cameras-28` subview.
- Camera cards use `camera_view: auto`, not live. Five live streams eat bandwidth. Live doorbell video also crashes the Fire WebView (Chrome 104). The `cameras` subviews on kitchen, lounge and wall-panel still have `live` on Front Door and Garden (unverified whether they crash).
- Every Ring camera card on `lounge-dash`, `office-dash` and `hall-dash` taps to `intent:#Intent;package=com.ringapp.fire;end` (all cameras, not just the doorbell: Zero, 4 Nov 2026). Other dashboards: doorbell card only; `wall-panel` doorbell cards have no intent tap. The tap opens the Ring app, not live video in the browser.
- The red smoke banner (`input_boolean.smoke_alarm_sounding`) sits at the top of the first view of kitchen, lounge, bedroom, office and wall-panel.
- Calendar cards: only `dayGridMonth`, `dayGridDay`, `listWeek` exist. There is no week grid.
- A bare `switch.` shows under no Lights. Wrap module switches with switch_as_x.

**Fully Kiosk settings that matter** (all five read 4 Nov 2026; Fully Plus licensed on all)

| Setting | Value | Note |
|---|---|---|
| `kioskMode` | true (Apolosign false) | Off = drops out of kiosk. Only takes effect after `restartApp`. `deviceInfo.kioskLocked` is the honest read. |
| `microphoneAccess` | true | Off = the mic prompt every time. |
| `kioskExitGesture` | 0 (swipe from left) | Never try 3; it left Zero locked in. Don't guess values. PIN is factory 1234. |
| `disableCamera` | true | All five true (Apolosign fixed 4 Nov 2026). |
| `motionDetection`, `webcamAccess`, `motionDetectionAcoustic` | false | Camera rule. |
| `kioskAppWhitelist` | `com.ringapp.fire` | The Ring app on Fire is `com.ringapp.fire`, not `com.ringapp`. Ring is installed on the Office, Lounge and Hall tablets (4 Nov 2026). The Hall tablet's App Store said "network unreachable"; it was installed from amazon.co.uk → Deliver to the tablet instead. |
| `enableUrlOtherApps` | true | Needed for the intent tap. |
| `enablePopups` | true | HA opens url actions as a new window. Off = "popups and new tabs disabled". |
| `injectJsCode` | Office only: forces HA light theme | HA theme is per browser. Use the same on any tablet that flips dark. |
| `timeToScreensaverV2` | 300 (Apolosign 0) | — |

- Do not send `restartApp` while another app (Ring) is in front. On `.190` it left port 2323 closed until Zero reopened Fully by hand (4 Nov 2026).
- The Apolosign's kiosk layout comes from `?kiosk` on its URL, not the dashboard's `kiosk_mode`. Any URL pushed to it must keep `?kiosk`.
- Other dashboards use `kiosk_mode.mobile_settings`: hide sidebar and header below 1000 px.
- New tablet setup: copy settings by diffing `listSettings` off an existing tablet; camera settings go first. Fire setup forces Amazon Kids; tap Back about 5 times quickly to get past it. Reserve its IP on the MR600.

---

## 6. Integrations and gotchas

- **Zigbee2MQTT pairing.**
  - Open `permit_join` over MQTT. For long windows, re-publish `{"time":254}` every 4 min.
  - ZBMINI with no installer: Zero flips the lighting breaker off for 10 s and back on; the module pairs on power-up. Don't tell him to press the module.
  - 06P: unplug and replug its USB.
  - Close permit_join after. Name the device, set its area, and set detach/power-on per section 3.
  - The pairing date is HA device_registry `created_at`, not the Z2M last-seen time.
- **DJT11LM cat flap.** It will not take a sensitivity write on this network; don't retry. The template `binary_sensor.kitchen_cat_flap_used` = vibration AND strength ≥ 15. It has not yet caught a real Sox trip.
- **TRVZB.** Fitted only where a plumber has put in a valve body. Each valve runs on its room sensor (external mode). Keep Wi-Fi off 2.4 GHz channel 1.
- **Tuya.**
  - Cloud entry `tuya` is linked to Zero's Tuya app account. tuya_local runs the Lounge (`.213`), Master Bedroom (`.106`) and Bedroom 2 (`.110`) fans locally. On each, the cloud light entity is disabled and the night light stays on cloud.
  - Find fans with `tinytuya.deviceScan` inside the HA container; a 6668 port scan misses them.
  - A half-finished tuya_local flow blocks re-adding (`already_in_progress`, hidden). Only an HA restart clears it.
  - Changing a fan's IP: options flow. Options override data.
- **Alexa.** Lights and fans come through the Tuya app's Alexa skill. `alexa_devices` gives Nova speech out of the Echos.
  - HA restarts have broken `alexa_devices`. The fix is ⋮ → Reconfigure with password plus authenticator OTP (SMS not accepted). Restarts on 6 and 10 Oct 2026 did not break it.
  - The Bedroom light still needs renaming to "Bedroom Light" in the Alexa app.
- **Ring.**
  - The account has the house cameras plus the 28 cameras. The house doorbell is the `_2` set.
  - If dings stop but `last_activity` still updates, the push listener is dead. `custom_components/fcm_padding_fix` patches firebase-messaging 0.4.5 padding; remove it when upstream ships #37/#43. A dead listener after network blips is fixed by reloading the Ring entry (`ring_push_watchdog`).
  - Ring motion switches died on upstream bug core#178622 (3 Sep 2026). Unverified whether that bug is still present.
  - The doorbell drops off Wi-Fi when its battery is low.
- **Lounge TV.**
  - ON = stick HOME (`input keyevent 3`) over CEC. A bare WAKEUP leaves the screen dark.
  - OFF = `remote.send_command Standby` on `remote.lounge_tv`. The Philips always reads "off" in HA (`/6/powerstate` 404).
  - Never remove either Fire entity. The old two-entity rule (built-in = power, stick = apps) came from the JVC. The Philips now takes the built-in's power role.
  - Philips Eco "Picture mute" blanks the screen; a volume key brings it back. Ambilight in standby is the TV's own setting, for Zero to turn off.
- **PS5.**
  - `ps5-mqtt` handles rest and wake only. Launching a game is impossible via playactor by design; don't re-research it.
  - When running playactor by hand, add `NODE_OPTIONS=--require /app/ps5-mqtt/server/dist/playactor-preload.js`.
  - The local skip-Login patch must be re-applied if the image updates. The PSN npsso expires about 7 Dec 2026.
  - Hisense control uses the `vidaa_tv` custom component with a local HDMI2 patch in `coordinator.py`. Code changes need a full HA restart. The TV's MQTT port 36669 is a dead end.
- **Fire Sticks.** If an "Allow USB debugging" loop appears, unplug the stick for 30 s. `adb_allowed_connection_time` is 0. Never send Zero into Settings while a dialog blocks the screen.
- **Voice PE / Hey Nova.**
  - Custom microWakeWord model served from `/local/wakeword/hey_nova.*`, cutoff 0.98.
  - If a speaker goes deaf, check `select.*_wake_word` = Hey Nova and `select.*_assistant` = Nova first.
  - After flashing, reload the esphome entry.
  - Voice "stops working": check `docker logs whisper` for empty transcripts before blaming Nova.
  - Editing `custom_components/nova/conversation.py` needs a full HA restart.
  - Live 4 Nov 2026: Kitchen is up. Bedroom (`.153`) and Office (`.198`) are unavailable. The 4th unit is not in HA.
- **Hue (historical).** No Hue in the house since 8 Oct 2026. If a bulb is ever needed: factory reset from the Hue Bluetooth app. Deleting it doesn't reset it, and touchlink is impossible here.
- **Cats / Tractive.** `device_tracker.sox`, `device_tracker.marshmellow`. Hardware QCGTANTA = Sox, VGDLXEGO = Marshmellow. Each tracker needs its own premium subscription. The CARTO watermark on maps is HA core's.
- **hOn washing machine** (`custom_components/hon`): cloud only. Not in HA config entries on 4 Nov 2026 (unverified whether it was ever configured).
- **Hildebrand Glow:** installed via HACS. No config entry on 4 Nov 2026.
- **Printer:** Canon TS4100i via ipp at `.115`, reserved on the MR600 3 Nov 2026.
- **HA updates:** `docker compose pull homeassistant && docker compose up -d homeassistant` in `~/projects/homeassistant/`. Proxy settings live in `.storage/http`, not YAML.

---

## 7. Still open / waiting on hardware

| Item | Last true |
|---|---|
| 3 × Fire HD 8 on order (summer house, Bedroom 2, Bedroom 3). 1 × Fire HD 8 to buy (Garage). | 31 Oct 2026 |
| 2 × FULEN lights and a solar Ring camera to buy for the summer house. | 31 Oct 2026 |
| Presence sensors to place: Garage, hall, landing, Bedroom 3. | 31 Oct 2026 |
| Plumber to fit TRV valve bodies. Bedroom 2 valve skipped (bed). 6th head not found. | 3 Nov 2026 |
| Garage CHANFOK fan into the Tuya app (pairing failed 29 Oct). Garage module mode Edge vs the Detach rule — Zero to decide. | 4 Nov 2026 |
| `climate.garage_heater` unavailable; no dew-point automation. | 4 Nov 2026 |
| Optimum OP-WFSTAT thermostat ordered. Add a boiler thermostat card to hall-dash once in. | 3 Nov 2026 |
| Broadband: router goes in the Lounge. Then re-check `ping -c 30 192.168.1.213` and re-make the reservations. Elysium onto its wired point. | 3 Nov 2026 |
| Ring app installed only on the Office tablet; install `com.ringapp.fire` on the others. | 4 Nov 2026 |
| Office tablet still low while "plugged in" (11 % at 17:34). | 4 Nov 2026 |
| Bedroom and Office Voice PEs unavailable. Living Room Voice PE not in HA. | 4 Nov 2026 |
| Alexa: rename the bedroom light to "Bedroom Light" in the Alexa app. | 26 Oct 2026 |
| PS5 → TV automations not yet seen triggered by the controller. | 26 Oct 2026 |
| Lounge presence missed Zero sitting still once (24 Oct). Watch it. | 26 Oct 2026 |
| Marshmellow tracker battery 19 %. | 4 Nov 2026 |
| Still to come, not specced: smart meter (Glow), bathroom/kitchen extractor survey, room-level presence via ESP32 BLE proxies (try two first). | Install page |
| HA "Office" area still carries the alias "Summer House" while a separate Summer House area exists. | 4 Nov 2026 |

---

## 8. How to check before changing anything

All of these are reads. Never write config from the shell. `automations.yaml` is root-owned; changes go through the HA API.

```sh
# Token
T=$(docker exec nova printenv HA_TOKEN)

# All states / one entity
curl -s -H "Authorization: Bearer $T" http://192.168.1.146:8123/api/states | python3 -m json.tool | less
curl -s -H "Authorization: Bearer $T" http://192.168.1.146:8123/api/states/switch.lounge_fan_module_detach_relay_mode

# Module modes at a glance
curl -s -H "Authorization: Bearer $T" http://192.168.1.146:8123/api/states | python3 -c "import json,sys;[print(e['entity_id'],e['state']) for e in json.load(sys.stdin) if 'detach_relay_mode' in e['entity_id'] or e['entity_id'].endswith('power_on_behavior')]"

# One automation's config (writes: POST same path, then automation/reload)
curl -s -H "Authorization: Bearer $T" http://192.168.1.146:8123/api/config/automation/config/fan_modules_always_on

# Config files (root-owned)
sudo cat ~/projects/homeassistant/config/automations.yaml
sudo python3 -c "import json;[print(e['domain'],e['title'],e['data'].get('host'),e.get('options',{}).get('host')) for e in json.load(open('/home/coolzerohacks/projects/homeassistant/config/.storage/core.config_entries'))['data']['entries']]"
```

```sh
# Dashboards: websocket only (url_path uses hyphens; the .storage file uses underscores)
T=$(docker exec nova printenv HA_TOKEN) python3 -c "
import json,os,websocket
w=websocket.create_connection('ws://192.168.1.146:8123/api/websocket'); w.recv()
w.send(json.dumps({'type':'auth','access_token':os.environ['T']})); w.recv()
w.send(json.dumps({'id':1,'type':'lovelace/config','url_path':'hall-dash'})); print(json.dumps(json.loads(w.recv())['result'],indent=1))"
# Or read the file: sudo cat ~/projects/homeassistant/config/.storage/lovelace.hall_dash
# To SEE a dashboard: headless Chrome with localStorage.hassTokens (Fully screenshots miss WebGL/maps).
```

```sh
# Zigbee2MQTT config and device list (password comes from its own config; never print it)
docker exec zigbee2mqtt cat /app/data/configuration.yaml
P=$(docker exec zigbee2mqtt sh -c "grep -A4 '^mqtt:' /app/data/configuration.yaml | awk '/password:/{print \$2}'")
docker exec mosquitto mosquitto_sub -h localhost -u nova -P "$P" -t 'zigbee2mqtt/bridge/devices' -C 1 -W 5 | python3 -m json.tool | less
# Watch wall-switch presses live
docker exec mosquitto mosquitto_sub -h localhost -u nova -P "$P" -t 'zigbee2mqtt/+/action' -v
```

```sh
# Fully Kiosk (host + password per tablet live in core.config_entries, domain fully_kiosk)
curl -s -G "http://<host>:2323/" --data-urlencode cmd=deviceInfo   --data-urlencode type=json --data-urlencode "password=<pw>"
curl -s -G "http://<host>:2323/" --data-urlencode cmd=listSettings --data-urlencode type=json --data-urlencode "password=<pw>"
# The way out of kiosk that always works: http://<host>:2323 in a LAN browser.
```

```sh
# Elysium network
ip -br addr; ip route; journalctl -t wifi-watchdog --since today; journalctl -k --since -10min | grep mt7921u
```

Where secrets live: HA token in the `nova` container env (`Nova-2.0/.env`). MQTT password in Z2M `configuration.yaml`. Fully passwords in HA `core.config_entries`. Wi-Fi password in esphome `secrets.yaml`. Fully admin password also in the Pantheon vault (`FULLY_KIOSK_ADMIN_PASSWORD`); it is not the same on every tablet, so trust `core.config_entries`.

## 9. New tablet checklist (every Fire tablet; written 4 Nov 2026)

Set these on any new Fire tablet before calling it done. Values were read off live tablets. Set them through Fully's REST (`setBooleanSetting` / `setStringSetting`), then read them back with `listSettings`.

| Step / setting | Value | Why |
|---|---|---|
| Fire setup | Tap Back ~5 times fast to get past the forced Amazon Kids screen. | New Fire setup forces Amazon Kids even with no child profile. |
| Router reservation | Reserve its MAC on the MR600 (see `project_elysium_4g_wifi_dongle` memory for the login method). | Tablets use the IP, not the hostname (no Pi-hole). |
| Fully Plus licence | Licensed | Popups, kiosk, regain focus and global orientation are Plus features. |
| HA integration | Add `fully_kiosk` with host + remote admin password | Password then lives in `core.config_entries`. |
| `startURL` | `http://192.168.1.146:8123/<room>-dash/<view>` | |
| `kioskMode` | true (Apolosign: false, see section 6) | Off = drops out of kiosk. Takes effect after `loadStartUrl` or `restartApp`. Check `deviceInfo.kioskLocked`. |
| `kioskExitGesture` / `disableKioskExitGesture` | `0` / false | Left-edge swipe + PIN 1234 is the way out on the tablet. Gesture 3 does not work. |
| `microphoneAccess` | true | Off = mic prompt every time. |
| `disableCamera` true; `motionDetection`, `webcamAccess`, `motionDetectionAcoustic` false; HA `camera.<tablet>` disabled | | No tablet cameras (Zero: "creepy"). |
| Ring app | Install from **amazon.co.uk → Apps & Games → Ring → Deliver to <tablet>** if the tablet's App Store says "network unreachable" (Hall, 4 Nov 2026). | Doorbell/camera cards open the Ring app; live video crashes the Fire WebView. |
| `kioskAppWhitelist` | `com.ringapp.fire` | Ring's package on Fire. `com.ringapp` does nothing. |
| `enableUrlOtherApps` | true | Lets the `intent:` link leave Fully. |
| `enablePopups` | true | HA opens url actions as a new window. Off = "popups and new tabs disabled". |
| `forceScreenOrientation` | `3` (Reverse Portrait: power lead at the bottom) | Zero's choice, 4 Nov 2026. Values from Fully's own page: 0 Auto, 1 Portrait, 2 Landscape, 3 Reverse Portrait, 4 Reverse Landscape. Not on the Apolosign (landscape wall screen). |
| `forceScreenOrientationGlobal` | true | Makes the Ring app follow the orientation (it locks itself upright otherwise). |
| `timeToRegainFocus` | `120` | Set, but it did NOT bring Fully back on the Lounge tablet (4 Nov 2026). The real return is the HA automation `tablets_back_to_fully`: add the new tablet's `sensor.<tablet>_foreground_app` + `button.<tablet>_bring_to_foreground` to it. |
| `touchesOtherAppsBreakIdle` | true | Touching Ring restarts the 2 minutes, so it isn't pulled away mid-use. |
| Dashboard cameras | Every Ring camera card: `tap_action: url` → `intent:#Intent;package=com.ringapp.fire;end`; `camera_view: auto`. | |
| Smoke alert | Add the tablet to `smoke_co_alarm_phone_alert` (screen on + overlay + TTS, and the overlay clear at the end). | Hall added 4 Nov 2026 as `*_fire_tablet_*_2`. |

**Fully stops answering on port 2323 when another app (Ring) is in front on some tablets.** Don't send `restartApp` then: it can leave Fully closed until someone opens it by hand.
