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
| `esphome` | Voice PE firmware | Configs: `home-assistant-voice-09c7c5.yaml`, `-bedroom.yaml`, `-office.yaml`, `summer-house-voice.yaml` (14 Nov 2026; hostnames max 31 chars, so not `home-assistant-voice-summer-house`). |
| `matter-server` | Matter, host network | Nothing paired to it. |
| `whisper` | STT, `small-int8`, beam 2 | `base-int8` dropped whole sentences (12 Sep 2026). |
| `piper` | Local TTS, `en_GB-alan-medium` | Nova's own voice goes through Nova's `/api/tts`, not piper. |
| `ps5-mqtt` | PS5 rest/wake | `~/projects/ps5-mqtt/`, local playactor patch. |
| `pihole` | DNS for Elysium, cosmos-hq, cosmos laptop ONLY | Not the LAN's DNS, on purpose. |
| `nginx-proxy` | `*.cognitocoding.com` hostnames | Only resolve via Pi-hole. |
| `nova` | Nova; holds `HA_TOKEN` | Nova has `ha_*` MCP tools. |

**Network (9 Nov 2026)**
- Broadband live 9 Nov 2026. Router is **the EE hub** (call it that, never "BT hub"), gateway `192.168.1.254`, Hub Manager at `https://192.168.1.254` (self-signed cert). Admin password = the old MR600 one. SSID `EE-MX82JZ`, same password as before.
- EE hub DHCP range widened to `.20–.253` (default started at `.146`, which blocked the old reservations). 18 "Always use this IP" reservations set 9 Nov, matching the MR600's old addresses: .40 .97 .105 .106 .107 .110 .115 .122 .146 .163 .190 .191 .213 .221 .246 .247 .250. Set in Address table → device → "Always use this IP address". A page reload logs you out; navigate with the in-page tabs.
- Reserved since (static on the hub, read 14 Nov 2026): PS5 `.46`, PlayStation TV `.65`, Lounge Voice `.246`. Added 14 Nov 2026 at their current addresses: Summer House Voice `.72`, Kitchen Voice `.157`, Bedroom Voice `.99` (was `.153`; HA's esphome entry already follows `.99`), Bedroom TV Hisense `.244`. Not seen on the hub: `.215` (98:e7:f4:a7:78:75).
- **Reserving on the EE hub from Elysium:** Playwright (if the MCP is down, the `playwright` package in `~/.npm/_npx/e41f203b7505f1fb/node_modules/` works from a headless node script; pass the password in an env var), a new browser context with `ignoreHTTPSErrors:true` (self-signed cert), `https://192.168.1.254` → Advanced settings → password (the old MR600 admin password; feed it to the page from a throwaway `127.0.0.1` server, never paste it into code) → Address table. Wait about 8 s for the device list, click the button containing `MAC Address: <mac>`, flip `#toggle_switch` to YES, `#save`. To change the IP as well: `#single_device_ip_address_ip_address_button`, fill `#single_device_ip_address_key_3` with the last octet, then `#single_device_ip_address_ip_address_ppopup_set_btn`. Read back `Always use this IP address: YES`.
- Lounge fan `.213`: 0% loss over 30 pings on 9 Nov (was 27% on the 4G router).
- Before 9 Nov: TP-Link Archer MR600 (4G) with a Poynting XPOL-2-5 antenna, gateway `192.168.1.1`, same SSID.
- TP-Link RE505X extender added downstairs 3–4 Nov 2026, same SSID.
- MR600 2.4 GHz is pinned to channel 11 / 20 MHz, auto-channel off (3 Nov 2026). Channel 1 sat on top of Zigbee channel 11 and broke the Bedroom 3 valve. Set the RE505X and the future broadband router the same way.
- 21 DHCP reservations on the MR600 (3 Nov 2026). Admin login is through the `tplinkrouterc6u` lib, username `admin`.
- Elysium has no Wi-Fi of its own. It is online through an Alfa AWUS036AXM USB dongle (`mt7921u`, interface `wlx00c0cab9c757`). The dongle MUST sit in a black USB 2 rear port; on USB 3 it never comes up. Since 18 Nov 2026 the dongle is backup only: it takes a DHCP address (`.151` on 18 Nov), route metric 600, and the cable is primary.
- Elysium is in the Garage (Zero, 14 Nov 2026). Plan: a second **EE Smart WiFi booster** (ordered 14 Nov 2026) goes in the Garage, which is halfway between the hub and the summer house. Elysium gets wired into the booster's Ethernet port, which ends its reliance on the dongle, and the booster covers the summer house. On cutover: reserve `.146` on the EE hub for `eno1` MAC `c8:7f:54:5d:29:cd`, take the dongle out of the default route, and stop `wifi-watchdog` (it re-plugs the dongle and reboots on its rules).
- The Garage EE booster is live (seen 18 Nov 2026): `0a:f8:7e:37:5c:43` (2.4 GHz, channel 1) / `06:f8:7e:37:5c:42`, about -54 dBm at Elysium. Elysium's dongle joined it and gets the EE hub (`.254`) as gateway. Wired 18 Nov 2026: `eno1` has carrier at 1000 Mb/s, holds `.146` from the EE hub, and is the default route (metric 100, dongle 600); HTTPS out over `eno1` returns 200, and the dongle sits at -28 dBm on the booster. Cutover done 18 Nov 2026: the EE hub already had `.146` reserved for `eno1` (Address table → "Asus Device" → Always use this IP address: YES). The fixed `.146` was taken off the dongle in `/etc/netplan/60-wifi.yaml` (backup `60-wifi.yaml.bak-20261118`), live with `ip addr del 192.168.1.146/24 dev wlx00c0cab9c757`. Before that, `.146` sat on both interfaces and pings out over the cable got nothing back. After: 0% loss to `1.1.1.1` over `eno1`, HA on `.146:8123` answers, Voice PEs answer. `wifi-watchdog.timer` stopped and disabled.
- **A TP-Link "Archer" router outside the house (the MR600 is not in this house, Zero 18 Nov 2026) reaches Elysium at about -85 dBm.** 18 Nov 2026 16:41 it was seen broadcasting `EE-MX82JZ` with its own DHCP (gateway `192.168.1.1`, BSSID `b8:fb:b3:43:10:43`, web page says "Archer"). Elysium roamed onto it and lost every house device and the EE hub. Fix from Elysium: `sudo wpa_cli -i wlx00c0cab9c757 bssid_ignore b8:fb:b3:43:10:43` then `sudo wpa_cli -i wlx00c0cab9c757 roam <booster BSSID>`; check `ip route` shows `via 192.168.1.254`. The ignore is lost on a wpa_supplicant restart or a dongle re-plug.
- The upstairs EE booster is `6e:70:56:68:e5:5c` (2.4 GHz channel 1, IP `.206`; WPS says "Smart Hub SH31B"). It can't move closer to the summer house. Channel 1 overlaps Zigbee channel 11 (see the channel note below).
- `/usr/local/sbin/wifi-watchdog` (timer **disabled 18 Nov 2026**, now that Elysium is on the cable; re-enable with `sudo systemctl enable --now wifi-watchdog.timer` only if Elysium goes back on the dongle):
  - No internet for 3 min → USB re-plug of the dongle.
  - Kernel logs `mt7921u ... failed:-110` (driver wedged) → forced reboot, NEVER a re-plug. A re-plug of a wedged dongle froze Elysium at 15:58 on 4 Nov 2026.
  - A re-plug that hangs for 60 s → forced reboot.
  - Hub unreachable after a re-plug → reboot, at most once per 30 min.
  - 4G outage (hub answers, no internet) → re-plug every 15 min, never reboot.
  - Alerts go to ntfy topic `elysium-ht5nauq27ayyfma`.
  - `GW` in the script was still the MR600's `192.168.1.1` until 14 Nov 2026, so any broadband outage of 8+ min would have forced a reboot. Now `192.168.1.254`; backup `/usr/local/sbin/wifi-watchdog.bak-20261114`.
- `host-scripts/ha-health.sh`, cron 08:00 → ntfy: failed integrations, Repairs, batteries <20%, and (14 Nov 2026) updates. Zigbee firmware updates are installed by it every morning; HACS updates are only reported, because they need an HA restart and that breaks Alexa. `--print` is read-only.
- **Sleeping Zigbee sensors (SNZB-02 temperature, 02LD fridge/freezer) refuse `update.install`**: Z2M logs "Device didn't respond to OTA request". Fix: `mosquitto_pub -t zigbee2mqtt/bridge/request/device/ota_update/schedule -m '{"id":"<name>"}'` (Z2M 2.13). The sensor updates itself on its next wake. Undo with `.../ota_update/unschedule`. Done for all nine 14 Nov 2026. Mains devices (TRVZB) take `update.install` fine. Kitchen and Hall Temperature (SNZB-02B) were still on 4101 on 17 Nov 2026; Z2M logs `Handling of poll check-in ... genPollCtrl.checkinRsp failed` for exactly those two, so they wake but miss the reply. Re-scheduled 17 Nov 2026 (`state.json` → `update.state: scheduled`). If they are still on 4101 after that, the schedule is not the fix. 18 Nov 2026: still 4101 and still `scheduled`, while both report normally (battery 100 %, linkquality 84–114, readings every few minutes). So the schedule is not the fix; don't re-queue it again. They work fine on 4101; the update only improves battery life. 19 Nov 2026: both on 4103 (`update.kitchen_temperature` / `update.hall_temperature` installed = latest = 4103), so the schedule did get through in the end; give a stuck sleeper a few days before calling it failed.
- **HA errors are in `config/home-assistant.log`, not `docker logs`** (docker logs is empty). `sudo grep -E "ERROR|CRITICAL" ~/projects/homeassistant/config/home-assistant.log`.
- **A Repair that names a service the automation no longer uses is stale.** Check the automation config, then clear it with HA's own fix flow: `POST /api/repairs/issues/fix {"handler":"automation","issue_id":...}`, then `POST /api/repairs/issues/fix/<flow_id> {}`. Two smoke-alert ones cleared 14 Nov 2026.
- Zero's phone notify service is `notify.mobile_app_mobile` (renamed from `mobile_app_mobile_a57` when the app re-registered; the smoke alert was pointed at the old name until 14 Nov 2026). If a Repair says `service_not_found notify.mobile_app_*`, check `get_services` for the current name.

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
| Other | Sox's litter box (Tuya). PetSafe microchip flap in the back door. Voice PE **Kitchen Voice** (`kitchen_voice`, `.157`, 14 Nov 2026), live on Hey Nova. |
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
| Voice PE | **Lounge Voice** = the old Kitchen unit `home_assistant_voice_09c7c5` (`.246`), moved to the Lounge/office desk (Zero, 14 Nov 2026). HA device renamed and set to Living Room; entity IDs kept as `*_09c7c5_*`. Unmuted 14 Nov 2026. Wake word sensitivity raised to **Moderately sensitive** 14 Nov 2026 (Zero: "doesn't pick up very well"); the others stay Slightly. Whisper heard it clearly whenever it did wake. |

### Master Bedroom
| | |
|---|---|
| Fixtures | CHANFOK 16" fan + light, tuya_local `.106` (Yunlong profile; the Chanfok profile wants DP 53, which this unit never sends). 1-gang 1-way rocker. |
| Module | `Master Bedroom Fan Module`. **Detach**, power-on `on`. Same reason as the Lounge. |
| Sensors | Master Bedroom Presence (06P), Master Bedroom Temperature (02B). |
| Heat | 20 °C at 06:00, 14 °C at 09:00 — he sleeps cold. |
| Tablet | `.247` → `bedroom-dash`. Charges from the bed's USB; no socket, no module. |
| TV | Bedroom Fire Stick 4K (`.250`). Install page puts the JVC Fire TV Edition here. HA still lists the JVC built-in as `media_player.lounge_fire_tv` (`.221`, unavailable) in Living Room; unverified. `media_player.bedroom_eris_tv_bedroom_tv` is unavailable. |
| Voice PE | `bedroom_voice` (`.99`, reserved). Zero unplugged it in early Nov because it went off in the night (cause unknown; history gone). Plugged back in 14 Nov 2026: Hey Nova, Nova, Slightly sensitive, aggressive. If it fires at night again, get the time from Zero and read `docker logs -t whisper` for what it heard. |

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
| Heat | Bedroom 3 Radiator (TRVZB, on the room sensor). **Gets no hot water (16 Nov 2026).** Checked by Zero: valve pin under the Sonoff head moves freely; lockshield (brass spindle, new plastic pipe side) turned fully open; both pipes cold to the floor. The plumber moved it without lifting floorboards: plastic pipe along the skirting, a plain grey push-fit elbow behind the bed (no valve in it). 17 Nov 2026: the boiler ran 06:25–08:20 (hall valve 15 → 19.5 °C) while the Bedroom 3 valve called at 26 °C, and the room stayed flat at 12.7–13.2 °C. So no flow even with the boiler firing = air trapped in the new pipe. Bled 17 Nov 2026 (Zero): water straight away, no air in the radiator. So the blockage is in the new pipe run or the valve body, not the radiator. Waits for the same plumber (off sick 17 Nov 2026); Zero doesn't want a different one. 17:25 same day: hall valve back at 19.5 °C (boiler firing), Bedroom 3 room still 13.3 °C. How to check from Elysium: room-temperature history of `sensor.hall_radiator_local_temperature` shows when the boiler fired; compare `sensor.bedroom_3_temperature_temperature` over the same window. |
| Tablet | On order. `bedroom-3-dash` exists. |

### Hall and stairs
| | |
|---|---|
| Fixtures | Two plain LED lights. Bottom plate is 2-gang 2-way; top plate is 1-gang 2-way. |
| Modules | `Stairs Bottom Light`, `Stairs Top Light`. **Edge.** switch_as_x lights. |
| Why | One module per light, so the house can switch the bottom one alone (the doorbell-after-dark automation). No Hue anywhere since 8 Oct 2026. |
| Sensors | Front Door (04PR2), Hall Temperature (02B). Hall and landing presence still to place. |
| Safety | Ei3016 smoke alarms in the hall and on the landing. Ei3018 CO alarm downstairs by the boiler. All on radio interlink. Ei428 relay → `Smoke Alarm Relay` ZBMINI (detach on, reports only), area Hall. |
| Heat | Hall Radiator valve in **auto**, running the same program as the Optimum wall thermostat every day (16 Nov 2026): 05:15 19.5 °C, 09:30 18, 16:30 19, 22:30 14. Change both together. Drops to 5 °C while the front door is open. Cold radiator while its smart valve shows 100 % open and calling = lockshield (capped valve, other end) closed: open it anticlockwise fully. The hall one was seized shut until 16 Nov 2026. |
| Tablet | `.122` → `hall-dash`. Fully Plus licensed 3 Nov 2026. Locked upside-down (power lead at the bottom, Zero's choice, 4 Nov 2026): `forceScreenOrientation=3` (Reverse Portrait, value read off Fully's own settings page) + `forceScreenOrientationGlobal=true` so the Ring app follows. |

### Garage
| | |
|---|---|
| Fixtures | CHANFOK fan/light (29 Oct 2026), on module `Garage Light`, detach **off** (Edge). The fan is not in the Tuya app yet. It came in via the Chanfok Smart app and has to be re-paired in Tuya. |
| Conflict | The rule says a smart fitting → Detach. The install page says "if a smart fitting goes in it becomes Detach". Live is Edge. Unresolved: do not change it without Zero. |
| Sensors | Garage Temperature (02WD sealed), Garage Back Door Sensor (04PR2), Garage Door up-and-over (04P). Presence still to place. |
| Heat | Devola wall heater with its own frost stat (Zero set it to anti-frost mode 16 Nov 2026), on a permanently live fused spur (Always On, no module). `climate.garage_heater` is unavailable. |
| Boiler | The Potterton combi is in the Garage (Zero, 16 Nov 2026), and so is Elysium. Shows E1 lockouts. The display only ever shows `E1`, nothing more (Zero, 17 Nov 2026). On Potterton, E1 = the boiler reads low pressure. 16 Nov 2026: hit 3 bar hot after cold top-ups, so the relief valve (copper pipe → tundish → garage drain, not the condensate) dumps water and pressure falls again. Expansion vessel suspected: Gas Safe engineer. Until then: top up only to 1 bar cold, never more. 17 Nov 2026: E1 several times in the day with the gauge at 1 bar (Zero), so not low pressure. Not a frozen condensate either: outside 6.8–7 °C, Garage 6.6–7.9 °C. E1 with the gauge at 1 bar = the boiler's own pressure sensor disagrees with the gauge. Zero's plumber is also his Gas Safe engineer (off sick 17 Nov 2026); one visit covers the E1, the expansion vessel and Bedroom 3. No other engineer (Zero). Until then: reset on E1, gauge at 1 bar. |
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
- Ring at the house: Front Door doorbell (`_2` entities, Doorbell 3 Plus), Backdoor, Path, Garden, Lounge Pet Cam, and `camera.office_live_view` (named "Summer House Live view"). That last one IS the summer house camera: its snapshot shows the summer house inside (9 Nov 2026).
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
| Boiler: Potterton combi, run by the Optimum wall thermostat and its own RF receiver at the boiler. The thermostat's program is what fires the boiler. No HA or Tuya link to the receiver, and none is needed. The hall valve runs the same program. Settled 16 Nov 2026: don't suggest a new thermostat, Hive, a Sonoff thermostat or a ZBMINI on the boiler. | Zero's call. | 16 Nov 2026 |

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
| `kitchen_radiator_back_door` / `hall_radiator_front_door` | Door open → save the setpoint in `input_number.kitchen_radiator_saved_setpoint` / `input_number.hall_radiator_saved_setpoint` (never saves 5) → valve to 5 °C. Door shut 10 s, HA start, or every 10 min: door shut and valve still on 5 °C → put the saved setpoint back. | Don't heat the street. Until 18 Nov 2026 it used a `scene.create` snapshot, which stuck the kitchen on 5 °C twice that day: a 1 s close→open flicker snapshotted the 5, and an HA restart with the door open lost the snapshot (door came back `unknown→off`, not `on→off`). Tested 18 Nov: kitchen set to 5 °C with the door shut, back to 18.5 °C at the next 10-min tick. Backup `automations.yaml.bak-20261118-doors`. |
| `hall_ / kitchen_ / living_room_ / master_bedroom_ / bedroom_3_radiator_external_sensor` | Writes the room sensor into the valve's external temperature input on change, every 30 min and at HA start. | The valve heats to the room, not to the radiator. |
| `lounge_radiator_daytime` / `lounge_radiator_night` | 10:00 → 18 °C, 22:00 → 16 °C. | Zero's schedule. |
| `master_bedroom_radiator_schedule` | 06:00 → 20 °C, 09:00 → 14 °C. | He sleeps cold. |
| `kitchen_radiator_schedule` | 06:00 → 17 °C, 22:00 → 14 °C. | Zero's schedule. |
| `ring_push_watchdog` | Log "Shutting down push receiver" → reload the Ring entry after 2 min. | firebase_messaging gives up after 3 network errors. Needs `system_log: fire_event: true`, which is in config and loaded since HA started 16:22 on 4 Nov 2026. Not triggered yet. |

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
| Office (Zero's desk, Lounge) | `.190` | `.../office-dash/office` | Official Raspberry Pi USB-C supply (5 Nov 2026). Never an extension-lead USB port or the PC's USB: both gave too little power and it drained while "plugged in" (5 Nov: 8→5 % in 20 min). |
| Hall | `.122` | `.../hall-dash/hall` | Unverified |
| Kitchen Apolosign 21.5" | `.40` | `.../kitchen-dash/kitchen?kiosk` | Mains |
| Summer House | `.176` (reserved 9 Nov 2026) | `.../summer-house-dash/summer-house` | Unverified. Set up 9 Nov 2026: checklist settings applied and read back, HA entry "Summer House Tablet" (`*_fire_tablet_*_3`, camera disabled, area Summer House), in the smoke alert. **Fully Plus not licensed yet** (watermark; kiosk still locks, `kioskLocked` true after HA login). Restoring Fire OS from another tablet's backup does not carry Fully settings or the HA login over. |
| Bedroom 2, Bedroom 3 | — | On order | — |
| Garage | — | To buy | — |

`wall-panel` (Main DashBoard), `tv-remotes` and `map` are not on any tablet's start URL.

**What each dashboard must and must not show (Zero's rules)**
- Rooms are separate **dashboards, not tabs** inside one. Built wrong once, 18 Aug 2026.
- `lounge-dash`: lounge light and fan only (never other rooms' lights), TV/stick, PlayStation section, thermostat. The pet cam was removed from the Lounge (18 Aug and 31 Oct 2026); do not put it back. The Garage cam is in its cameras subview.
- `office-dash`: title "Office", lounge light + fan, diary, front/back/garden cameras, lounge temp/humidity/presence, thermostat. **Never the TV or PS5.**
- `bedroom-2-dash` / `bedroom-3-dash`: **deleted 4 Nov 2026 (Zero: "scrap them").** No tablet pointed at them. Backups of the storage files are in `~/lovelace.bedroom_[23]_dash.bak-20261104`. Do not recreate them.
- `summer-house-dash` (9 Nov 2026): Front door (doorbell camera → Ring app, front door contact, last ring), Summer house (temperature, humidity, door. **No summer house camera** (Zero: not needed when he is in there, 9 Nov 2026), hourly weather. Smoke banner in the header. Built so Zero can watch the front door from the summer house.
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
- New tablet setup: copy settings by diffing `listSettings` off an existing tablet; camera settings go first. Fire setup forces Amazon Kids; tap Back about 5 times quickly to get past it. Reserve its IP on the EE hub.

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
  - `ha-health.sh` line "28 Marshmallow Battery" is the Ring **camera** at 28 Courageous, not Marshmellow's Tractive tracker (that one is `sensor.marshmellow_tracker_battery`, 95 % on 17 Nov 2026).
- Daily-run noise, not faults (16 Nov 2026): hundreds of `firebase_messaging ... Unexpected exception during read` a day while `event.*_motion` entities still update (push alive), and `Invalid value for number.*_radiator_timer_mode_target_temp: 0` from the TRVZBs. Only act on Ring if motion/ding events stop.
- **Lounge TV.**
  - ON = stick HOME (`input keyevent 3`) over CEC. A bare WAKEUP leaves the screen dark.
  - OFF = `remote.send_command Standby` on `remote.lounge_tv`. The Philips always reads "off" in HA (`/6/powerstate` 404).
  - Never remove either Fire entity. The old two-entity rule (built-in = power, stick = apps) came from the JVC. The Philips now takes the built-in's power role.
  - Philips Eco "Picture mute" blanks the screen; a volume key brings it back. Ambilight in standby is the TV's own setting, for Zero to turn off.
- **Philips lounge TV (`custom_components/philips_js`, 9 Nov 2026).** A copy of HA 2026.9.1's core integration with one patch in `coordinator.py`. The 55OLED760 has no `/powerstate` (404) and `/screenstate` always says "Off", so stock HA showed the TV off for good. The patch reads Chromecast port 8008 instead: open = on, closed = standby (both measured on the set 9 Nov). On an HA upgrade, re-copy core `philips_js` and re-apply the patch, or delete the folder if upstream fixes it.
- **PS5.**
  - `ps5-mqtt` handles rest and wake only. Launching a game is impossible via playactor by design; don't re-research it.
  - When running playactor by hand, add `NODE_OPTIONS=--require /app/ps5-mqtt/server/dist/playactor-preload.js`.
  - The local skip-Login patch must be re-applied if the image updates. The PSN npsso expires about 7 Dec 2026.
  - Hisense control uses the `vidaa_tv` custom component with a local HDMI2 patch in `coordinator.py`. Code changes need a full HA restart. The TV's MQTT port 36669 is a dead end.
- **Fire Sticks.** If an "Allow USB debugging" loop appears, unplug the stick for 30 s. `adb_allowed_connection_time` is 0. Never send Zero into Settings while a dialog blocks the screen.
- **Voice PE / Hey Nova.**
  - **Adding a NEW unit — do exactly this, nothing else (14 Nov 2026: I sent Zero to the phone app, then a website, then Bluetooth, before finding this):**
    1. Copy `esphome/config/home-assistant-voice-office.yaml` to `<room>-voice.yaml`, change `name` (≤31 chars) and `friendly_name`, compile in the `esphome` container.
    2. Zero plugs the unit into **Elysium's USB** with a USB-C cable. It shows as `/dev/ttyACM0`.
    3. Flash with a throwaway container that has `--device /dev/ttyACM0` (the `esphome` container has no device mapping). Wi-Fi is baked into the firmware, so no app, website or Bluetooth onboarding.
    4. He moves it to the room. HA discovers it on Wi-Fi → add it, set area, `select.*_wake_word` = Hey Nova, `select.*_assistant` = Nova, reserve the IP on the EE hub.
    - Never route a new unit through the HA app, the Voice PE website or HA Bluetooth: Elysium has no BlueZ, and that is not how any unit here was done.
  - Custom microWakeWord model served from `/local/wakeword/hey_nova.*`, cutoff 0.98.
  - If a speaker goes deaf, check `select.*_wake_word` = Hey Nova and `select.*_assistant` = Nova first.
  - After flashing, reload the esphome entry.
  - Voice "stops working": check `docker logs whisper` for empty transcripts before blaming Nova.
  - **Wake word fires but replies say "I only caught 'you'" or nothing (empty `stt-no-text-recognized`) → weak Wi-Fi, not the mic or Nova.** Read the unit's signal: `esphome logs` fails with "Connection requires encryption" because HA holds a `noise_psk` for every unit, so use `aioesphomeapi` inside the `homeassistant` container with the `noise_psk` from `core.config_entries` (`subscribe_logs(..., dump_config=True)` prints `Signal strength` and `BSSID`). Kitchen reads -42 dB and works. The Summer House unit read -75 dB on the EE hub's BSSID on 14 Nov 2026, logged `api.connection ... Socket operation failed errno=11`, and only one go out of several worked. 19 Nov 2026 (Garage booster live): still on the upstairs booster `6E:70:56:68:E5:5C`, channel 6, -74 dB, not the Garage booster. Its three requests that morning were all heard, no errors. Same day: `summer-house-voice.yaml` now lists the Garage booster BSSID `0A:F8:7E:37:5C:43` first (priority 10), with a plain SSID fallback (backup `.bak-20261119`). OTA from the esphome container: `docker exec esphome esphome compile summer-house-voice.yaml` then `docker exec esphome esphome upload summer-house-voice.yaml --device 192.168.1.72`. After: Garage booster, -66 dB, ping 3–18 ms (was 150–220 ms). Pipeline history: websocket `assist_pipeline/pipeline_debug/list` + `/get`.
  - **Kitchen Voice** (14 Nov 2026): second Kitchen unit, `kitchen-voice.yaml`, `.157`, MAC `20:f8:3b:09:ba:25`, -49 dB on the EE hub. Entities `*.kitchen_voice_*`, area Kitchen, Hey Nova / Nova, finished speaking `aggressive` like the others. A new ESPHome entry defaults that to `default`, so set it. Added over REST: `POST /api/config/config_entries/flow {"handler":"esphome"}` then `{"host":...,"port":6053}`. IP not yet reserved on the EE hub. The old Kitchen unit (`09c7c5`) is now Lounge Voice.
  - The Summer House unit sits on the old **Office Voice** HA entry (host `.72`), so its entities are `*.office_voice_*` with friendly names "Summer House Voice ...".
  - Editing `custom_components/nova/conversation.py` needs a full HA restart.
  - Live 4 Nov 2026: Kitchen is up. Bedroom (`.153`) and Office (`.198`) are unavailable. The 4th unit is not in HA.
- **Hue (historical).** No Hue in the house since 8 Oct 2026. If a bulb is ever needed: factory reset from the Hue Bluetooth app. Deleting it doesn't reset it, and touchlink is impossible here.
- **Cats / Tractive.** `device_tracker.sox`, `device_tracker.marshmellow`. Hardware QCGTANTA = Sox, VGDLXEGO = Marshmellow. Each tracker needs its own premium subscription. Battery `0` + `sensor.*_tracker_status` = `not_reporting` comes from Tractive's cloud: the tracker is flat or off. Reloading the tractive entry does not change it (tested 15 Nov 2026); it needs charging. The CARTO watermark on maps is HA core's.
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
| Router reservation | Reserve its MAC on the EE hub (Address table → device → Always use this IP). | Tablets use the IP, not the hostname (no Pi-hole). |
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
| `timeToRegainFocus` | `120` | Fully comes back to the front 2 min after the **last touch** (its idle clock is `lastUserInteractionTime`; touches in Ring restart it because `touchesOtherAppsBreakIdle=true`). Proven on the Lounge tablet 20:45 4 Nov 2026: flipped back at idle 126 s. It did not fire once earlier that evening, so a backup job runs on Elysium: `/usr/local/sbin/tablets-back-to-fully`, root crontab every minute, same rule (other app + 2 min since last touch → `toForeground`), logs to `journalctl -t tablets-back-to-fully`. It reads hosts from HA's fully_kiosk entries, so a new tablet is covered once it is in HA. |
| `touchesOtherAppsBreakIdle` | true | Touching Ring restarts the 2 minutes, so it isn't pulled away mid-use. |
| Dashboard cameras | Every Ring camera card: `tap_action: url` → `intent:#Intent;package=com.ringapp.fire;end`; `camera_view: auto`. | |
| Smoke alert | Add the tablet to `smoke_co_alarm_phone_alert` (screen on + overlay + TTS, and the overlay clear at the end). | Hall added 4 Nov 2026 as `*_fire_tablet_*_2`; summer house 9 Nov 2026 as `*_fire_tablet_*_3`. |

| `launchOnBoot` | true | Was false on all four Fire tablets until 4 Nov 2026, so Fully did not come back after a restart. |
| `keepScreenOn` + `keepScreenOnAdvanced` | true + true | The screen must never really switch off: idle = Fully screensaver (black), tap = dashboard. Every tablet has a Fire OS passcode, so a real screen-off lands on the Amazon passcode screen. Plain `keepScreenOn` only holds while Fully is in front; on 5 Nov 2026 the Office tablet had Ring open, Fire OS switched the screen off at 14:35 and it came back on the passcode. Advanced added to all four Fire tablets 5 Nov 2026. |

**Fire OS kills Fully (4 Nov 2026, proven from Fully's own log via remote admin `home?cmd=showLog`).** A kill shows as "Start app ..." with no "Exit app" before it; a crash shows "Restart app after Uncaught Crash" and Fully restarts itself. Kills seen: Office 16:03/16:05/16:26, Hall 19:50 (Ring open), Lounge ~20:51 (6 min after Ring was left in the background). After a kill the tablet sits on the Fire home screen because `com.amazon.firelauncher` is still the default launcher, and nothing relaunches Fully. The real fix is making Fully the launcher (`adb shell pm disable-user --user 0 com.amazon.firelauncher`), which needs Developer Options + Enable ADB + a USB cable to Elysium once per tablet (adb is installed on Elysium). Zero: "all too much", 4 Nov 2026 — not done. Don't push it again unless he raises it.

**Fully stops answering on port 2323 when another app (Ring) is in front on some tablets** (that is Fully being killed, see above). Don't send `restartApp` then: it can leave Fully closed until someone opens it by hand.
