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
   {% set idle_state = printer.idle_timeout.state | string %}
   {% set e_target = printer.extruder.target | float | round(1) %}
   {% set bed_target = printer.heater_bed.target | float | round(1) %}
   {% set msg_only_once = printer["gcode_macro ACTIVATE_POWER_OFF"].msg_only_once | int %}
   {% if msg_only_once == 0 %}
      M118 Printer shutdown is active: Waiting for the extruder to cool down to 50°C, then the printer will be turned off.
      SET_GCODE_VARIABLE MACRO=ACTIVATE_POWER_OFF VARIABLE=msg_only_once VALUE=1  
   {% endif %}  
   {% if idle_state == "Idle" or idle_state == "Ready" %}
      {% if printer.extruder.temperature < 50.0 and printer.heater_bed.temperature < 50.0 %}
         {% if e_target == 0.0 and bed_target == 0.0 %}
            UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
            SET_GCODE_VARIABLE MACRO=ACTIVATE_POWER_OFF VARIABLE=msg_only_once VALUE=0
            _POWER_OFF_PRINTER
         {% endif %}	
      {% elif e_target == 0.0 and bed_target == 0.0 %}
         UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=2
      {% elif e_target > 0.0 or bed_target > 0.0 %}
         UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
         M118 The scheduled printer shutdown has been canceled: Temperature interaction by the user.
      {% else %}
         UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
      {% endif %}	  
   {% elif idle_state == "Printing" %}
      UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0
      M118 The scheduled printer shutdown has been canceled: User-side print interaction.	  
   {% endif %}
```

The only thing left is to add the following macro to your print end gcode. This will run the entire verification procedure to ensure it is safe to turn off the printer (the same macro is used for the button):
```
ACTIVATE_POWER_OFF
```

And that is it. Now after your print has finished, the printer will cooldown and once the temperature is below 50 degrees Celsius it will self-shutdown.

# Post Print Start Power OFF checks:
In order to achive this you would need to add the following Macros:
```
[gcode_macro ACTIVATE_POWER_OFF]
variable_msg_only_once: 0
gcode:
   SET_GCODE_VARIABLE MACRO=ACTIVATE_POWER_OFF VARIABLE=msg_only_once VALUE=0
   UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=60

[gcode_macro DEACTIVATE_POWER_OFF]
gcode:
   UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=0
   UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=0

[delayed_gcode POWER_OFF_PRINTER_CHECK_ACT]
gcode:
   {% set idle_state = printer.idle_timeout.state | string %}
   {% if (idle_state == "Idle" or idle_state == "Ready") and printer.print_stats.state != "paused" %}
      UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK DURATION=30
   {% else %}
      UPDATE_DELAYED_GCODE ID=POWER_OFF_PRINTER_CHECK_ACT DURATION=60
   {% endif %}
```
By clicking running Activate Power Off macro while the printer is running the macro will check every 60 seconds if the printer has finished. 
