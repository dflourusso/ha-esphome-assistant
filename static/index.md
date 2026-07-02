# DFLTech Assistant

ESPHome firmware for an ESP32-S3 voice assistant using an INMP441 I2S microphone and MAX98357 I2S amplifier.

## First-time installation

Connect the ESP32-S3 via USB, then use the button below to flash the factory firmware from your browser (Chrome or Edge required).

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

<esp-web-install-button manifest="firmware/manifest.json">
  <button slot="activate">Install DFLTech Assistant</button>
  <span slot="unsupported">Your browser does not support WebSerial. Use Chrome or Edge on desktop.</span>
  <span slot="not-allowed">HTTPS is required (or use localhost).</span>
</esp-web-install-button>

## After flashing

1. The device starts a Wi-Fi access point named `dfltech-assistant-XXXXXX` (password: `dfltech-setup`).
2. Connect with your phone; the captive portal opens automatically (or go to http://192.168.4.1/).
3. Enter your home Wi-Fi credentials.
4. Add the device in Home Assistant via the ESPHome integration (`dfltech-assistant-XXXXXX.local`).
5. Configure a Home Assistant Assist pipeline with speech-to-text and text-to-speech.
6. Wire a doorbell button between GPIO10 and GND, or use the **Start Conversation** button in Home Assistant.

Firmware updates are offered automatically in Home Assistant when a new release is published.

## Factory reset

Hold the **BOOT** button (GPIO0) on the ESP32-S3 for **10 seconds**, then release. Wi-Fi credentials are erased and the setup access point starts again.
