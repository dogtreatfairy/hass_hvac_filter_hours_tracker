# HVAC Filter Hours Tracker for Home Assistant Ecobee Integration
This project was primarily built to solve an issue with the way Ecobee tracks filter hours. It should track from the last date the filter was replaced, but it seems to track the hours from the last time the filter reminder was reset regardless of the actual filter change date. Home Assistant does not have the HVAC Fan On/Off status implemented as a sensor, so this also adds that sensor. 

## Step 1:
In `sensors.yaml` Add a custom sensor template to track HVAC Fan On/Off state.  
Replace `climate.home` with your actual HVAC Entity ID. 
```
- platform: template
  sensors:
    hvac_fan:
      friendly_name: "HVAC Fan"
      value_template: >
        {% if 'fan' in state_attr('climate.home', 'equipment_running') %}
          On
        {% else %}
          Off
        {% endif %}
    hvac_fan_filter_usage_percentage:
      friendly_name: "HVAC Fan Filter Usage Percentage"
      value_template: >
        {% set total_hours = states('input_number.hvac_filter_change_hours') | float %}
        {% set fan_on_hours = states('sensor.hvac_fan_on_time') | float %}
        {% if total_hours > 0 %}
          {% set percentage = ((total_hours - fan_on_hours) / total_hours) * 100 %}
          {{ (percentage | float | round(1)) }}
        {% else %}
          0
        {% endif %}
      unit_of_measurement: "%"
      icon_template: "mdi:percent"
  - platform: history_stats
    name: "HVAC Fan On Time"
    entity_id: sensor.hvac_fan
    state: "On"
    type: time
    start: "{{ states('input_datetime.hvac_filter_changed') }}"
    end: "{{ now() }}"
    unit_of_measurement: "h"

```
## Step 2:
In `configuration.yaml` add a custom date_time and input number: 

```
input_datetime:
  hvac_filter_changed:
    name: "HVAC Filter Changed"
    has_date: true
    has_time: true
    icon: "mdi:air-filter"

input_number:
  hvac_filter_change_hours:
    name: "HVAC Filter Change Hours"
    min: 10
    max: 1000
    step: 1
    unit_of_measurement: "hours"
    mode: box
```

## Step 3: 
You can optionally add a configuration panel to your lovelace configuration like this: 

<img src="https://github.com/user-attachments/assets/0071e12a-ec59-4d21-bbc5-16cf3e30589f" width="50%" />


### The Reset Button can be made using mushroom cards using this yaml:
```
type: custom:button-card
name: Reset HVAC Filter
tap_action:
  action: call-service
  service: script.reset_hvac_filter
  confirmation:
    text: Are you sure you want to reset the filter change time?
    confirm_button: "Yes"
    dismiss_button: "No"
styles:
  card:
    - background-color: rgb(120, 0, 0)
  name:
    - color: white
    - font-weight: bold
```

### And this script named `reset_hvac_filter`:
```
action: input_datetime.set_datetime
metadata: {}
target:
  entity_id: input_datetime.hvac_filter_changed
data:
  timestamp: "{{ now().timestamp() }}"
```
