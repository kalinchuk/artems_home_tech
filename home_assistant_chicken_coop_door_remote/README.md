# Home Assistant Chicken Coop Door Remote

A lot of motorized chicken coop doors on the market come with a 433MHz remote control. The device outlined below can be used to control the chicken coop door remotely via Home Assistant and can also automate the door opening and closing. The full assembly and instructions are available on [YouTube](...).

<img width="700" alt="Home Assistant Chicken Coop Door Remote" src="https://github.com/user-attachments/assets/12ebbce2-7308-4dcf-ae36-486efc21d041"/>

## What is Home Assistant

Home Assistant is a free and open-source software used for home automation. If you're not familiar with it, check out [https://www.home-assistant.io/](https://www.home-assistant.io/)

## How it works

An ESP32 is flashed with software capable of sending data using a 433MHz radio (see below). The software can also receive data using the receiver (see below). Home Assistant has a module called [ESPHome](https://esphome.io) which does all the configuration for the ESP32 devices and integration into Home Assistant. Once configured, the device will show up in Home Assistant.

<img width="506" alt="Home Assistant Chicken Coop Door Switch" src="https://github.com/user-attachments/assets/521b9254-bda3-48d7-a5cc-a3092a16bdd8" />

In essence, this device is a 433MHz transceiver for Home Assistant and can be used to control any number of 433MHz devices or receive data on that frequency for Home Assistant to consume and act on.

## Parts

| Part | Source |
|------|--------|
| ESP32 | [Amazon](https://www.amazon.com/dp/B0D9LJN2VW) |
| 433MHz transmitter | [Amazon](https://www.amazon.com/dp/B01DKC2EY4) |

The Amazon link to the 433MHz transmitter also includes a receiver which you can use as well.

## Schematic

<img width="915" alt="Home Assistant Chicken Coop Door Remote Schematic" src="https://github.com/user-attachments/assets/e46f9444-1074-4dc2-8d71-424ee788e9c1" />

## Configuration

To configure, first add a new device in ESPHome.

### ESPHome Device

In ESPHome, configure the ESP32 with the base configuration (esphome, esp32, api, ota, wifi). These values are automatically configured when a new device is added in ESPHome.

```yaml
esphome:
  name: 433mhz-transceiver
  friendly_name: 433MHz Transceiver

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "SOMEKEY"

ota:
  - platform: esphome
    password: "SOMEPASSWORD"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "433Mhz-Transceiver"
    password: "SOMEPASSWORD"

captive_portal:
```

Then, configure the transmitter for the chicken coop door. Also, define a cover that will be used as a trigger for opening and closing the door.

```yaml
remote_transmitter:
  pin: GPIO19
  carrier_duty_percent: 100%

cover:
  - platform: template
    name: Chicken Coop Door
    device_class: door
    open_action:
      - remote_transmitter.transmit_rc_switch_raw:
          code: '111111111000111110100010'
          protocol:
            pulse_length: 300
          repeat:
            times: 10
            wait_time: 0s
      - remote_transmitter.transmit_rc_switch_raw:
          code: '111111111000111110101000'
          protocol:
            pulse_length: 300
          repeat:
            times: 10
            wait_time: 0s
    close_action:
      - remote_transmitter.transmit_rc_switch_raw:
          code: '111111111000111110100010'
          protocol:
            pulse_length: 300
          repeat:
            times: 10
            wait_time: 0s
      - remote_transmitter.transmit_rc_switch_raw:
          code: '111111111000111110100100'
          protocol:
            pulse_length: 300
          repeat:
            times: 10
            wait_time: 0s
```

Notice that each code is sent multiple times (10 times in this case). This is to ensure that the chicken coop door receiver actually receives the signal since it's not very reliable.

*Note*: The `code` that is sent may be different in your case. To find out what code to use, you may need to listen to the signals while pressing the buttons on your 433MHz remote. Configure the device to listen with this configuration which can be added in the same file as the configuration above:

```yaml
remote_receiver:
  - id: RF_RCV
    pin:
      number: GPIO21
      mode: INPUT
      inverted: True
    tolerance: 60%
    filter: 255us
    idle: 4ms
    buffer_size: 2kb
    dump:
      - rc_switch
```

Adjust the tolerance and filter as needed to filter out noise.

Once added and installed on the device, the device logs will print out any messages received from the 433MHz receiver.

### Add Device to Home Assistant

Once the ESPHome device has been added and comes online, go to "Devices & Services" under "Settings" in Home Assistant and add this new device to Home Assistant. It needs to be added to Home Assistant directly because we defined a cover in the YAML configuration that needs to be available in Home Assistant.

### Configure Cover in Home Assistant

The final step is to configure a cover in Home Assistant that will be used as a switch to open and close the chicken coop door. This step is optional since you can just use the cover defined in the ESPHome device. The reason why you might want to add this additional cover is because you can configure a state for it using a separate sensor (i.e. contact sensor on the door).

In `configuration.yaml`, add the following:

```yaml
cover:
  - platform: template
    covers:
      chicken_coop_door:
        unique_id: chicken_coop_door_cover
        device_class: door
        friendly_name: Chicken Coop Door
        position_template: "{{ is_state('binary_sensor.chicken_coop_opening_2', 'on') }}"
        open_cover:
          - service: cover.open_cover
            target:
              entity_id: cover.433mhz_transceiver_chicken_coop_door_cover
        close_cover:
          - service: cover.close_cover
            target:
              entity_id: cover.433mhz_transceiver_chicken_coop_door_cover
```

Notice the `position_template` key. This is optional and is used for the state of the door. I'm using a simple ZigBee door sensor that I added to Home Assistant.

## Final Product

I soldered the ESP32 and the 433MHz receiver/transmitter to a protoboard and installed it in a basic enclosure. The antenna is for WiFi only. The power is supplied by a USB cable.

IMAGE

In Home Assistant, I added the new cover to the list of doors.

<img width="471" alt="Home Assistant Doors" src="https://github.com/user-attachments/assets/01d8b2c1-bbd8-475a-a525-97aedf672730" />

I also configured Home Assistant to open the door in the morning and close it in the evening using the "Automations" configuration.
