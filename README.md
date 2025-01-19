# ESPHome LYWSD02 Clock Synch
Example of how to synch the clock of an LYWSD02 with ESPHome and HomeAssistant.

Based on [this issue](https://github.com/esphome/feature-requests/issues/643).

# Home Assistant
Call the service with HomeAssistant to synch the clock
```yaml
action: esphome.esp_ble_gateway_update_time_to_ble_clock_sala
```
The connection and the time synch take some time but it should work. Check the logs in ESPHome to see the status of the connection.
# ESPHome Template
```yaml
esphome:
  name: esp-ble-gateway
  friendly_name: esp-ble-gateway

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "YOUR_ENCRYPTION_KEY"

  services:
  - service: update_time_to_ble_clock_sala
    then:
      - switch.turn_on: lywsd02_ble_connection

ota:
  - platform: esphome
    password: "YOUR_OTA_PASSWORD"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
    
bluetooth_proxy:
  active: true

time:
  - platform: homeassistant
    id: homeassistant_time

switch:
  - platform: ble_client
    id: lywsd02_ble_connection
    ble_client_id: lywsd02_clock
    restore_mode: ALWAYS_OFF

ble_client:
  - mac_address: E7:2E:01:61:E0:20 #Set the MAC address of your clock here
    id: lywsd02_clock
    on_connect:
      then:
        - lambda: |-
            ESP_LOGD("ble_client_lambda", "Connected to lywsd02_clock!");
        - delay: 3s
        - ble_client.ble_write:
            id: lywsd02_clock
            service_uuid: ebe0ccb0-7a0a-4b0c-8a1a-6ff2997da3a6 
            characteristic_uuid: EBE0CCB7-7A0A-4B0C-8A1A-6FF2997DA3A6
            value: !lambda |-
                auto time = id(homeassistant_time).utcnow();
                uint8_t time_byte1 =  time.timestamp & 0x000000ff;
                uint8_t time_byte2 = (time.timestamp & 0x0000ff00) >> 8;
                uint8_t time_byte3 = (time.timestamp & 0x00ff0000) >> 16;
                uint8_t time_byte4 = (time.timestamp & 0xff000000) >> 24;
                uint8_t timezone_byte = time.timezone_offset() / 1800;
                return {time_byte1, time_byte2, time_byte3, time_byte4, timezone_byte};

        - delay: 5s
        - switch.turn_off: lywsd02_ble_connection
    on_disconnect:
      then:
        - lambda: |-
            ESP_LOGD("ble_client_lambda", "Disconnected from lywsd02_clock!");

```
