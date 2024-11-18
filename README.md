# esphome-junctek_kgf
Component for esphome to read status from a Junctek KG-F coulometer/battery monitor via UART

## Features
Connects to the Junctek KGF series battery monitor via UART (RS-485 adapter needed) and retrieves the following values:
* Battery Voltage
* Battery Percent
* Current Amps
* Temperature

## Requirements
* ESPHome

## Tested setup
Tested on ESP32 using a RS-485 uart into a Junctek KG110F, but should work on an ESP8266 and any of the KG-F series

## Usage
### Connect hardware.
The ESP32 needs to be connected via an RS-485 module to the RS-485 on the monitor using a 4cp4 connector.

## ESPHOME Config
The applicable config for the device should look something like:

```yaml
uart: 
 ######### UART for the Juncteck (1) ######
  - id: uart0
    tx_pin: GPIO03
    rx_pin: GPIO01
    baud_rate: 115200
    rx_buffer_size: 384

sensor:
  - platform: junctek_kgf    
    uart_id: uart0
    invert_current: true
    voltage:
      name: "junctek1_battery_voltage"
    current:
      name: "junctek1_battery_current"
    battery_level:
      name: "junctek1_battery_level"
    temperature:
      name: "junctek1_temperature"
    ah_battery_level:
      name: "junctek1_ah_battery_level" 
    ah_total_used:
      name: "junctek1_ah_total_used"
    wh_battery_level:
      name: "junctek1_wh_battery_level"
    running_time:
      name: "junctek1_running_time"
    battery_internal_resistor:
      name: "junctek1_battery_internal_resistor" 
    battery_life:
      name: "junctek1_battery_life"
    power:
      name: "junctek1_power"  
    relay_status:
      name: "junctek1_relay_status" 
    direction:   
      name: "junctek1_direction"  
######## Relay set value ##########
    over_voltage_set:   
      name: "junctek1_over_voltage_set"  
    under_voltage_set:
      name: "junctek1_under_voltage_set" 
    positive_overcurrent_set:
      name: "junctek1_positive_overcurrent_set" 
    negative_overcurrent_set:
      name: "junctek1_negative_overcurrent_set" 
    over_power_protection_set:
      name: "junctek1_over_power_protection_set" 
    over_temperature_set:
      name: "junctek1_over_temperature_set" 
    protection_recovery_seconds_set:
      name: "junctek1_protection_recovery_seconds_set" 
    delay_time_set:
      name: "junctek1_delay_time_set" 
    battery_amphour_capacity_set:
      name: "junctek1_battery_amphour_capacity_set" 
    voltage_calibration_set:
      name: "junctek1_voltage_calibration_set" 
    current_calibration_set:
      name: "junctek1_current_calibration_set" 
    temperature_calibration_set:
      name: "junctek1_temperature_calibration_set" 
    reserved_set:
      name: "junctek1_reserved_set" 
    relay_normally_open:
      name: "junctek1_relay_normally_open" 
    current_ratio_set:
      name: "junctek1_current_ratio_set" 

```

UPDATED!!  all sensors added.
Address is assumed to be 1 if not provided. (this is configured on the monitor)
invert_current: This inverts the reported current, it's recommended to include this option with either true or false (which ever makes the current make more sense for your setup). The default is currently false (and false will match previous behaviour), but may change to true in future updates.
## Future work
More sensors/statistics are possible, as is adjusting various configuration, but haven't currently been added. File an issue if there anything you want to see.
