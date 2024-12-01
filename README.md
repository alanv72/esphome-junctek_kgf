# esphome-junctek_kgf
Component for esphome to read status from a Junctek KG-F coulometer/battery monitor via UART

## Features
Connects to the Junctek KGF series battery monitor via UART (RS-485 adapter needed) and retrieves the following values:
* Battery Voltage
* Battery Percent
* Current Amps
* Temperature
* Plus Much more!

## Requirements
* ESPHome

## Tested setup
Tested on ESP32 using a RS-485 uart into a Junctek KG110F, but should work on an ESP8266 and any of the KG-F series

## Usage
### Connect hardware.
The ESP32 needs to be connected via an RS-485 module to the RS-485 on the monitor using a 4cp4 connector.
-----------------------------------
alanv72 confirmed working with ESP8266 with rs485 that doesn't need flowcontrol ping like max485. Couldn't make work with esp32 or max485 ¯\_(ツ)_/¯ Working with a KG160F (600amp) with display connected.

expanded sensors based on https://github.com/tfyoung/esphome-junctek_kgf/discussions/14

Thanks to https://github.com/Lukylic!

Update c+ code for newest firmware that has different string for reading measurements and doesn't have power in Watt's. C+ code calcs Watts based on amp * volatage and reports with orginal sensor name.

2024/11/23 - Corrected Total AH calcs.

Sample:
Address/ID	CRC	Volts *.01	Amps *.01	AH Remaining *.001	AH Cumulative *.001	Watthours remaing	System runtime	Temp	Special Function	Relay state	Current Direction	Battery runtime in mins	Batt Resistance *01 mOh
5	105	5206	6054	295606	5353	1295308	14069	127	0	0	0	292	5499
5	141	5206	6056	295303	5656	1295308	14087	127	0	0	0	292	5515
5	140	5206	6055	295303	5656	1295308	14087	127	0	0	0	292	5515

Expanded sample with 10min rolling avg battery runtime:
[expanded_sample.yaml](https://github.com/alanv72/esphome-junctek_kgf/blob/main/expanded_sample.yaml)

## ESPHOME Config
The applicable config for the device should look something like:

```yaml
substitutions:
  name: master-batt-bus
  device_description: "Master Battery Bus Monitor - KG160F"

external_components:
   - source: github://alanv72/esphome-junctek_kgf@accum
     refresh: always
  
esphome:
  name: ${name}

esp8266:
  board: nodemcuv2
  restore_from_flash: true

# Enable logging
logger:
  level: INFO

# Enable Home Assistant API
api:
  encryption:
    key: "xxxxxxxx"

ota:
  - platform: esphome
    password: "xxxxxxx"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Master-Batt-Bus Fallback Hotspot"
    password: "xxxxxx"

captive_portal:

globals:
  - id: sample_1
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_2
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_3
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_4
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_5
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_6
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sample_index
    type: int
    restore_value: no
    initial_value: '0'
  - id: mean_battery_life_1min
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: mean_battery_life_10min
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: sum_1min_averages
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: count_1min_averages
    type: int
    restore_value: no
    initial_value: '0'
  - id: values_10min
    type: std::array<float, 10>  # Array to store last 10 values
    restore_value: no
    initial_value: 'std::array<float, 10>{}'
  - id: oldest_value_index
    type: int
    restore_value: no
    initial_value: '0'

interval:
  - interval: 10s
    then:
      - lambda: |-
          int index = id(sample_index);
          if (index >= 6) {
            index = 0;
          }

          float battery_life = id(mainbus_battery_life).state;
          float current_battery_level = id(mainbus_ah_battery_level).state;
          ESP_LOGI("mainbus_battery_ah_logger", "Main Bus Battery AH Remaining: %.2f", id(mainbus_ah_battery_level).state);

          // Check for NaN and handle it
          if (isnan(battery_life)) {
            ESP_LOGE("battery_life_error", "Received NaN value for battery life");
            battery_life = 0.0; // Set a default or retry fetching the value
          }

          // Store the current battery life in the current sample (in minutes)
          switch (index) {
            case 0: id(sample_1) = battery_life; break;
            case 1: id(sample_2) = battery_life; break;
            case 2: id(sample_3) = battery_life; break;
            case 3: id(sample_4) = battery_life; break;
            case 4: id(sample_5) = battery_life; break;
            case 5: id(sample_6) = battery_life; break;
          }

          // Update the sample index
          id(sample_index) = index + 1;

          // Calculate the mean battery life in minutes, including zeros if battery level is low
          float sum = 0.0;
          int valid_samples = 0;
          if (id(sample_1) >= 0 || current_battery_level < 10.0) { sum += id(sample_1); valid_samples++; }
          if (id(sample_2) >= 0 || current_battery_level < 10.0) { sum += id(sample_2); valid_samples++; }
          if (id(sample_3) >= 0 || current_battery_level < 10.0) { sum += id(sample_3); valid_samples++; }
          if (id(sample_4) >= 0 || current_battery_level < 10.0) { sum += id(sample_4); valid_samples++; }
          if (id(sample_5) >= 0 || current_battery_level < 10.0) { sum += id(sample_5); valid_samples++; }
          if (id(sample_6) >= 0 || current_battery_level < 10.0) { sum += id(sample_6); valid_samples++; }

          if (valid_samples > 0) {
            // Smoothing out fluctuations with EMA
            float alpha = 0.1; // Smoothing factor
            id(mean_battery_life_1min) = alpha * (sum / valid_samples) + (1 - alpha) * id(mean_battery_life_1min);
          } else {
            id(mean_battery_life_1min) = 0.0; // Handle case where all samples are zero
          }
          // ESP_LOGI("mean1min_debug", "Sum: %.2f, Valid Samples: %d, Mean 1min: %.2f", sum, valid_samples, id(mean_battery_life_1min));

  - interval: 1min
    then:
      - lambda: |-
          ESP_LOGI("mainbus_battery_life_logger", "Main Bus Battery Life: %.2f minutes", id(mainbus_battery_life).state);
          ESP_LOGI("sample_1_logger", "sample 1: %.2f minutes", id(sample_1));
          ESP_LOGI("sample_2_logger", "sample 2: %.2f minutes", id(sample_2));
          ESP_LOGI("sample_3_logger", "sample 3: %.2f minutes", id(sample_3));
          ESP_LOGI("sample_4_logger", "sample 4: %.2f minutes", id(sample_4));
          ESP_LOGI("sample_5_logger", "sample 5: %.2f minutes", id(sample_5));
          ESP_LOGI("sample_6_logger", "sample 6: %.2f minutes", id(sample_6));
          ESP_LOGI("mean1min_logger", "mean 1min: %.2f minutes", id(mean_battery_life_1min));

          // Update the rolling 10-minute average
          if (isnan(id(mean_battery_life_1min))) {
            ESP_LOGE("mean_battery_life_error", "Received NaN value for mean battery life 1min");
            id(mean_battery_life_1min) = 0.0;
          }
          
          if (id(count_1min_averages) < 10) {
            id(values_10min)[id(count_1min_averages)] = id(mean_battery_life_1min);
            id(sum_1min_averages) += id(mean_battery_life_1min);
            id(count_1min_averages)++;
          } else {
            id(sum_1min_averages) -= id(values_10min)[id(oldest_value_index)];
            id(values_10min)[id(oldest_value_index)] = id(mean_battery_life_1min);
            id(sum_1min_averages) += id(mean_battery_life_1min);
            id(oldest_value_index) = (id(oldest_value_index) + 1) % 10;
          }

          if (id(count_1min_averages) > 0) {
            id(mean_battery_life_10min) = id(sum_1min_averages) / id(count_1min_averages);
          } else {
            id(mean_battery_life_10min) = 0.0; // Handle case where no valid averages
          }
          // ESP_LOGI("mbl10t_logger", "MBL10T: %.2f", id(mean_battery_life_10min));

          // Calculate and log mean battery life in human-readable format
          float mean_minutes = id(mean_battery_life_10min);
          int days = static_cast<int>(mean_minutes) / 1440;
          mean_minutes = fmod(mean_minutes, 1440);

          int hours = static_cast<int>(mean_minutes) / 60;
          mean_minutes = fmod(mean_minutes, 60);

          int minutes = static_cast<int>(mean_minutes);
          ESP_LOGI("mean_battery_life_10min_logger", "Mean Battery Life Over 10 Minutes: %d days %d hours %d minutes", days, hours, minutes);

uart:
  - id: kg_uart
    tx_pin: 15
    rx_pin: 13
    baud_rate: 115200
    rx_buffer_size: 384
    stop_bits: 1
    data_bits: 8
    debug:
      direction: BOTH  # Log both RX and TX
      dummy_receiver: false  # If you want to see received data in logs
      after: 
        delimiter: "\n"  # Log after a newline if your protocol uses it

sensor:
  - platform: junctek_kgf
    uart_id: kg_uart
    address: 5
    invert_current: false
    voltage:
      name: "mainbus_battery_voltage"
    current:
      name: "mainbus_battery_current"
    battery_level:
      name: "mainbus_battery_level"
    temperature:
      name: "mainbus_temperature"
    ah_battery_level:
      name: "mainbus_ah_battery_level"
      id: mainbus_ah_battery_level
    ah_total_used:
      name: "mainbus_ah_total_used"
    wh_battery_level:
      name: "mainbus_wh_battery_level"
    running_time:
      name: "mainbus_running_time"
      id: mainbus_running_time
    battery_internal_resistor:
      name: "mainbus_battery_internal_resistor"
      unit_of_measurement: "mΩ"
    battery_life:
      name: "mainbus_battery_life"
      id: mainbus_battery_life
    power:
      name: "mainbus_power"
    relay_status:
      name: "mainbus_relay_status"
    direction:
      name: "mainbus_direction"
      id: mainbus_direction
    over_voltage_set:   
      name: "mainbus_over_voltage_set"
    under_voltage_set:
      name: "mainbus_under_voltage_set"
    positive_overcurrent_set:
      name: "mainbus_positive_overcurrent_set"
    negative_overcurrent_set:
      name: "mainbus_negative_overcurrent_set"
    over_power_protection_set:
      name: "mainbus_over_power_protection_set"
    over_temperature_set:
      name: "mainbus_over_temperature_set"
    protection_recovery_seconds_set:
      name: "mainbus_protection_recovery_seconds_set" 
      unit_of_measurement: "s"
    delay_time_set:
      name: "mainbus_delay_time_set" 
      unit_of_measurement: "s"
    battery_amphour_capacity_set:
      name: "mainbus_battery_amphour_capacity_set" 
    voltage_calibration_set:
      name: "mainbus_voltage_calibration_set" 
    current_calibration_set:
      name: "mainbus_current_calibration_set" 
    temperature_calibration_set:
      name: "mainbus_temperature_calibration_set" 
    reserved_set:
      name: "mainbus_reserved_set" 
    relay_normally_open:
      name: "mainbus_relay_normally_open" 
    current_ratio_set:
      name: "mainbus_current_ratio_set"
    avg_daily_ah_used:
      name: "mainbus_avg_daily_ah_used"
      id: mainbus_avg_daily_ah_used
    estimated_runtime:
      name: "mainbus_estimated_runtime"
      id: mainbus_estimated_runtime

text_sensor:
  - platform: template
    name: "Main Bus Monitor Run Time"
    id: mainbus_battery_runtime_friendly
    update_interval: 5 minutes
    lambda: |-
      float running_time_state = id(mainbus_running_time).state;

      // Check for NaN and handle it
      if (isnan(running_time_state)) {
        ESP_LOGE("runtime_error", "Received NaN value for running time");
        return std::string("Invalid data");
      }

      int total_seconds = static_cast<int>(running_time_state);

      int years = total_seconds / 31536000;  // 365 days/year for simplicity, ignoring leap years
      total_seconds %= 31536000;

      int days = total_seconds / 86400;
      total_seconds %= 86400;

      int hours = total_seconds / 3600;
      total_seconds %= 3600;

      int minutes = total_seconds / 60;

      std::string buffer = "";

      if (years > 0) {
        buffer += std::to_string(years) + " years ";
      }
      if (days > 0) {
        buffer += std::to_string(days) + " days ";
      }
      if (hours > 0) {
        buffer += std::to_string(hours) + " hours ";
      }
      buffer += std::to_string(minutes) + " mins";

      return buffer;

  - platform: template
    name: "Mean Battery Life Over 10 Minutes"
    id: mean_battery_life_10min_text
    update_interval: 1min
    lambda: |-
      float mean_minutes = id(mean_battery_life_10min);
      if (isnan(mean_minutes)) {
        ESP_LOGE("mean_battery_life_error", "Received NaN value for mean battery life");
        return std::string("Invalid data");
      }

      int total_minutes = static_cast<int>(mean_minutes);
      int days = total_minutes / 1440;
      total_minutes %= 1440;
      int hours = total_minutes / 60;
      int minutes = total_minutes % 60;

      std::string buffer = "";
      
      if (days > 0) {
        buffer += std::to_string(days) + " days ";
      }
      if (hours > 0) {
        buffer += std::to_string(hours) + " hours ";
      }
      buffer += std::to_string(minutes) + " mins";

      return buffer;
```

UPDATED!!  all sensors added.
Address is assumed to be 1 if not provided. (this is configured on the monitor)
invert_current: This inverts the reported current, it's recommended to include this option with either true or false (which ever makes the current make more sense for your setup). The default is currently false (and false will match previous behaviour), but may change to true in future updates.

