---
title: "Athom PG04V2"
date-published: 2026-11-26
type: plug
standard: uk
board: esp8266
difficulty: 1
---

The devices I have are marked "PG04V2 - AU16A" and are British plug standards. They contain a CSE7766 energy monitoring chip, not the HLW8032 that Athom specify here as of 2026-11-26: https://www.athom.tech/blank-1/uk-v2-plug-for-esphome
I think that I have older hardware.

## Basic Configuration
This exposes all energy monitoring features and the button and relay to HA. It also has a "Relay Lock" feature, which prevents any changes to the relay state either via the button or via HA. This is ideal for appliances such as fridges and freezers. Button pushes are still delivered to HA even when the relay is locked, so can be used for automation.

```yaml
# Board: Athom Smart Plug PG04V2-AU16A (UK plug)

substitutions:
  device_name: athom-smart-plug
  friendly_name: Athom Smart Plug
  # area: 
  activity_device_class: running
  main_icon: power-socket-uk
  default_state: RESTORE_DEFAULT_ON
  activity_threshold: 10 # Show as "Running" if drawing more than 10 W

esphome:
  name: "${device_name}"
  friendly_name: "${friendly_name}"
  area: "${area}"
  on_boot:
    priority: 600  # After state restoration, before network initialisation
    then: # recall relay lock status from flash
    - lock.template.publish:
        id: relay_lock
        state: !lambda |-
          return id(relay_locked)
            ? LOCK_STATE_LOCKED
            : LOCK_STATE_UNLOCKED;

esp8266:
  board: esp01_1m
  restore_from_flash: true

logger:
  baud_rate: 0 # prevent interference with UART
  level: INFO
  
api:
  encryption:

ota:
  - platform: esphome
    encryption:
      
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${friendly_name[:13]} Fallback Hotspot"
    password: !secret hotspot_wifi_password
    
captive_portal:

globals:
  - id: relay_locked
    type: bool
    restore_value: true
    initial_value: 'true'

lock:
  - platform: template
    name: Relay Lock
    id: relay_lock
    entity_category: config
    optimistic: true
    lock_action:
      - globals.set:
          id: relay_locked
          value: 'true'
    unlock_action:
      - globals.set:
          id: relay_locked
          value: 'false'

# Device Specific Config

uart: # Needed for CSE7766
  id: cse_uart_bus
  rx_pin: GPIO3
  baud_rate: 4800
  parity: EVEN
  stop_bits: 1

binary_sensor:
  # Push Button (Toggles relay when pressed, if relay is unlocked.)
  # Button pushes are still passed to HA even if the relay is locked.
  - platform: gpio
    pin:
      number: GPIO5
      inverted: true
    name: Button
    on_press:
      - switch.toggle: protected_relay
        
  - platform: status
    name: Server Status
      
  - platform: analog_threshold
    name: Activity State
    device_class: "${activity_device_class}"
    sensor_id: power
    threshold: ${activity_threshold}
    
switch:
  # Physical Relay
  - platform: gpio
    pin: GPIO12
    id: relay
    internal: true # cannot be directly controlled, for locking purposes
    restore_mode: "${default_state}"
    # ensure sync with virtual relay which is exposed to HA
    on_turn_on:
      - switch.template.publish:
          id: protected_relay
          state: ON
    on_turn_off:
      - switch.template.publish:
          id: protected_relay
          state: OFF

  # Protected Relay Control
  - platform: template
    name: "${friendly_name}"
    icon: "mdi:${main_icon}"
    id: protected_relay
    restore_mode: DISABLED
    turn_on_action:
      - if:
          condition:
            lock.is_unlocked: relay_lock
          then:
            - switch.turn_on: relay
    turn_off_action:
      - if:
          condition:
            lock.is_unlocked: relay_lock
          then:
            - switch.turn_off: relay
              
status_led: # Blue
  pin:
    number: GPIO13
    inverted: True
    
### Red LED is hardwired to relay status

sensor:
  - platform: uptime
    name: Uptime
    icon: "mdi:clock-outline"

  - platform: wifi_signal
    name: WiFi Signal

  # Power Monitoring  
  - platform: cse7766
    uart_id: cse_uart_bus
    apparent_power:
      name: Apparent Power
      filters:
        - throttle_average: 60s
    current:
      name: Current
      filters:
        - throttle_average: 60s
    energy:
      name: Energy
      unit_of_measurement: kWh
      accuracy_decimals: 3
      filters:
        - multiply: 0.001
        - throttle: 60s
    power:
      name: Power
      id: power
      filters:
        - throttle_average: 10s
    power_factor:
      name: Power Factor
      filters:
        - throttle_average: 60s
    reactive_power:
      name: Reactive Power
      filters:
        - throttle_average: 60s
    voltage:
      name: Voltage
      filters:
        - throttle_average: 60s
```
