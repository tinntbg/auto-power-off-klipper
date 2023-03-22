# Description
This is just a guide on how to configure automatic power off for a klipper 3d printer after a print has finished and the bed and toolhead are cooled. I couldn’t find a guide for this, so I decided to write one. 

To start with you would need to add a switch to your printer power. I'm using a Tasmota device, but you can use any device that you have. Here is a current list of available integrations for moonraker - gpio, klipper_device, rf, tplink_smartplug, tasmota, shelly, homeseer, homeassistant, loxonev1, smartthings, or mqtt. [Moonraker documentation](https://moonraker.readthedocs.io/en/latest/configuration/#power)

# Moonraker configuration 
First you would need to start by updating your moonraker configuration and adding this to the end of the file for a tasmota device, check moonraker documentation for configuring other options:
```
[power tasmota_plug]
type: tasmota
address: 192.168.0.12
password: ******
```

# Klipper configuration
Then we would need to configure klipper, by first testing connecting the wifi socket. I have personally created a hidden Macro for this:

```
[gcode_macro _POWER_OFF_PRINTER]
gcode:
  {action_call_remote_method("set_device_power",
                             device="tasmota_plug",
                             state="off")}
```

After that we would need to create a delayed gcode macro. In this macro will add logic that will check if both the bed and the hot end are below 50 degrees Celsius. If you start printing or start heating up the bed or hot end the automatic shutdown will be stopped. 
```
[delayed_gcode POWER_OFF_PRINTER_CHECK]
gcode:
  {% if printer.idle_timeout.state == "Idle" or printer.idle_timeout.state == "Ready" %}
    {% if printer.extruder.temperature < 50.0 and printer.heater_bed.temperature < 50.0 %}
        {% if printer.extruder.target == 0.0 and printer.heater_bed.target == 0.0 %}
            UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
            _POWER_OFF_PRINTER
        {% else %}
            UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=2
        {% endif %}
    {% else %}
        {% if printer.idle_timeout.state == "Printing" %}
            UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
        {% else %}
            {% if printer.extruder.target == 0.0 and printer.heater_bed.target == 0.0 %}
                UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=2
            {% else %}
                UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
            {% endif %}
        {% endif %}
    {% endif %}
  {% endif %}
```

The only thing left is to add the following command to your print end macro, that will set a 30 second timer:
```
UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=30
```

And that is it. Now after your print has finished, the printer will cooldown and once the temperature is below 50 degrees Celsius it will self-shutdown.

# Post Print Start Power OFF checks:
In order to achive this you would need to add the following Macros:
```
[gcode_macro ACTIVATE_POWER_OFF]
gcode:
    UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=60

[gcode_macro DEACTIVATE_POWER_OFF]
gcode:
    UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=0

[delayed_gcode POWER_OFF_PRINTER_CHECK_ACT]
gcode:
  {% if printer.idle_timeout.state == "Idle" or printer.idle_timeout.state == "Ready" %}
    UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=30
  {% else %}
    UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=60
  {% endif %}
```
By clicking running Activate Power Off macro while the printer is running the macro will check every 60 seconds if the printer has finished. 
