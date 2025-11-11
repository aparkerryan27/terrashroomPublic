# terrashroom

There's a set of headers next to the ESP32 designed for attaching a programmer harness. Check out the little PCB screenshot inside terrashroom board.png for a better sense of the physical object and where the pins are. 

Use this with a bed-of-nails / universal programmer board for ESP32, and some female header jacks, to flash new firmware to the main board.

As for the camera...... We will try to flash that seperately after breaking into the watertight camera compartment. Who knows

But we need SOFTWARE to flash onto the chamber board!
For this we will be using ESP Home.

## Repository layout (WIP)

```
esphome/                # ESPHome YAML + supporting scripts
  terrashroom.yaml      # active chamber firmware (ESPHome)
  secrets.example.yaml  # copy to a new secrets.yaml file with real creds
hardware/               # pinouts, wiring docs, physical notes
tests/                  # REST/OTA acceptance scripts (TODO)
```

### First-time ESPHome bring-up
- `cp esphome/secrets.example.yaml esphome/secrets.yaml` and set your Wi-Fi + OTA passwords.
- Install ESPHome (`brew OR pip install esphome`) if you have not already. You may also need esptool and some other programs, but execution failures should illuminate the missing links as you attempt to build.

# Use
To initially override the Terrashroom default code, plug the device into power (to support the heavy pull of the capacitors and such) and then
attach an ESP32 programmer (https://tinyurl.com/2em7ycs9) to the headers for EN, GPIO0, GND, TX, and RX. Do not attach the 5v pin, as that is already handled by the external power supply. Attach the bootloader to your device (you may potentially need to download drivers, such as CH34XVCPDriver https://learn.adafruit.com/how-to-install-drivers-for-wch-usb-to-serial-chips-ch9102f-ch9102/mac-driver-installation in order to see your device in your devices [ls /dev/cu*] list). 

When you find the device address, i.e. /dev/cu.wchusbserial120, run: 
esphome run esphome/terrashroom.yaml --device /dev/cu.wchusbserial120

While still plugged in to your computer, the logs should display the IP address of the device. This will be your web server for controlling the device remotely.

To upload Over The Air (OTA) after the initial setup of ESPHome, run:
esphome run esphome/terrashroom.yaml 
on a computer on the same wifi network as your device and the update will be sent to your device at its static IP automatically

### TODO

- Add documentation for humidity-driven control logic once those commits land.
- Populate `tests/` with scripts that curl the REST API and exercise OTA/safe-mode acceptance criteria.
- Stabilize the build toolchain. Either cache espressif32@6.12.0 locally or run once with unrestricted network so esphome run/OTA isn’t blocked. Likewise, pin PlatformIO in the YAML (platformio_options) to avoid random package upgrades mid-port.
- Long term: get the camera working, and an external server to store photos that will turn them into timelapses. 
- Implement native HDC3022 support. For now the firmware includes `includes/sensors_hdc1080.yaml`; if you have HDC3022 hardware you’ll need to swap to `includes/sensors_hdc3022.yaml` once the custom component stub there is filled in. In the meantime, the hdc1080 default from esphome appears to work fine to recieve info.


### GPIO Notes

- GPIO12 (heater MOSFET) and GPIO15 (water read enable) are ESP32 strapping pins. Avoid adding external pull-ups/pull-downs or large capacitances on these lines or the module may fail to boot. If you must probe them, disconnect power afterwards so the strapping levels return to ESP32 defaults before the next reset. THIS may be the source of failure that eventually prevented Parker's board from booting. This is being investigated.
- The water-level sensor lives on GPIO23 and is only powered when `Water Read Enable` (GPIO15) is high, so that we don't constantly run voltage through the water tank and heat the water or cause corrosion.


# Personal Notes for Plan Going Forward

Control Loop Plan

Introduce template numbers for humidity setpoint, CO₂/fan baseline, and temperature tolerance; expose them via the web UI so growers can tweak targets.
Draft pseudocode scripts: e.g., every 30 s compare chamber_humidity to setpoint; if below, bump atomizer/blower to Mode 2; if above, taper fan speeds—respecting the min duty constraints we enforced. Outside of the min-duty constraints tho, we can also just turn things off. Min-duty just helps us recognize that there is no difference between off and any smaller on values than min-duty. 
There is a very specific relationship between the atomizer and blower. The blower needs to be on high whenever the atomizer runs in order to properly spread the humid air throughout the chamber. The atomizer is a binary on/off, so the "value" of atomizer is actually a reference to how many seconds it should run in this "spritz" before we turn it off again. The blower also should run (not necessarily with the atomizer) about 15 seconds before the check of the humidity sensor every time on about 50% of max for 5 seconds, in order to distribute the humidity evenly around the chamber so that the sensor is actually reading the correct humidity. Because humidity changes take a fair bit of time to register, we want to increase the spritz time of the atomizer/blower really slowly (tiny incremements). We should wait a full minute at least between making a bump in the spritz time and triggering a spritz, before using the humidity values to modify spritz time and/or trigger another spritz.  COMMENT ALL OF THIS LOGIC DESCRIPTION IN TO THE CODE VERY WELL. This logic should follow a pretty basic PID setup with some boundaries for "acceptable range" with about 2 degrees of temperature (above and below) and 3% humidity (only limited below)

Add safety/time guards (max runtime, cooldown delays) and logging hooks so when we implement it in ESPHome automations we can trace decisions (e.g., “Humidity 5% low → Fan 2 @ 0.6”). Humidity should not ever get over 93%, if humidity is above 96%, turn the fans on full max to cycle fresh air through and rid the chamber of humidity that will be dangerous to the electronics. 

Mushroom Profiles Selector

Create a template select entity (ESPHome select:) listing species from Mushroom.h (Pink Oyster, Lion’s Mane, etc.) and store the choice in a global.
On selection change, set the humidity/temperature/CO₂ (fan speed) numbers to that mushroom’s recommended ranges; optionally trigger an info log so users know the profile applied.
Add a text_sensor (template) that surfaces the species description from Mushroom.h on the web UI, updating automatically when the selection changes, so growers see the target parameters and notes.


REST Test Scripts

Under tests/, add shell/python scripts that curl each REST endpoint (buttons, blower slider, switches) to verify HTTP 200 responses and correct state transitions (/light/<id>/turn_on, /switch/<id>/toggle, /number/<id>/set).
Include a smoke-test script that hits /json and asserts the presence of all sensors (temp/humidity/water level/IP) with non-empty values; fail fast if any key is missing.
Document how to run the suite (e.g., ./tests/rest_smoke.sh <device-ip>) and integrate it into future QA so firmware changes can be validated without the web UI.

Web/REST Documentation

Expand README with a “Device Interface” section listing every entity (lights, switches, numbers, sensors) and their REST endpoints, including example curl POST bodies for buttons/sliders.
Mention the static IP / mDNS name, fallback AP behavior, and how to retrieve /json for diagnostics; add screenshots or table summarizing the web UI layout.
Note any caveats (blower min, warnings about GPIO12/15 strapping pins, requirement to use HTTPS for remote access) so QA/support understands the constraints when replacing Blynk.
