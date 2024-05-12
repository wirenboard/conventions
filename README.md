## Wiren Board MQTT Conventions

The basic abstractions are *devices* and their *controls*. 

Each *device* has some *controls* assigned to it, i.e. parameters that can be controlled or monitored. *Devices* and *controls* are identified by names (arbitrary strings), and have some metadata. Metadata messages are published on device startup with `retained` flag set.
For example, some room lighting control *device* with one input (for wall switch) and one output (for controlling the lamp) *controls* is represented with MQTT topics as following:

* `/devices/RoomLight/meta` - JSON with all meta information about *device*
* `/devices/RoomLight/meta/error` - device-level error state, non-null means there was an error (usable as Last Will and Testament)
* `/devices/RoomLight/controls/Lamp` - contains current lamp state, '0' = off, '1' = on
* `/devices/RoomLight/controls/Lamp/on` - send a message with this topic and payload of '0'/'1' to turn lamp off or on
* `/devices/RoomLight/controls/Lamp/meta` - JSON with all meta information about control
* `/devices/RoomLight/controls/Switch` - contains current wall switch state
* `/devices/RoomLight/controls/Switch/meta` - JSON with all meta information about control
* `/devices/RoomLight/controls/Switch/meta/error` - non-null value means there was an error reading or writing the control. In this case  `/devices/RoomLight/controls/Switch` contains last known good value.

Each *device* usually represents the single physical device or one of the integrated peripheral of a complex physical device, although there are some boundary cases where the distinction is not clear. The small and not-so-complex real-world devices (say, wireless weather sensor) are ought to be represented by a single *device* in the MQTT hierarchy. 
Each *device* must be handled by a single driver or publisher, though it's not enforced in any way.

The *Conventions* are based on [HomA MQTT Conventions](https://github.com/binarybucks/homA/wiki/Conventions). The main changes are: no configuration is stored in MQTT (as MQTT is not so good as a database) and the *control* types system is more developed and complicated.

### Device's `/meta` topic

The topic contains all meta information in one JSON

```jsonc
{
    "driver": DRIVER_NAME,     // The name of a driver publishing the device
    "title": {
        "en": DEVICE_TITLE,    // English title of the device
        "ru": DEVICE_TITLE_RU, // Russian title of the device
        ...
    }
}
```
English title could be published in `/devices/+/meta/name` for backward compatibility with old conventions.


### Controls's `/meta` topic

The topic contains all meta information in one JSON

```jsonc
{
    // Control's type
    "type": "value",

    // Units. ASCII string. Could be set only for type "value". No units by default
    "units": "W",

    // Maximum allowed control's value. Default value for range type is 10^9, for other types no limit specified by default
    "max": 100,

    // Minimum allowed control's value. Default value for range type is 0, for other types no limit specified by default
    "min": -100.1,

    // Control's value is rounded to defined precision by a driver and it is also used during user input validation
    // If no precision is present, the value is used as-is
    "precision": 0.1,

    // Display order in user interface
    "order": 10,

    // The control doesn't have /on topic. Default value is false
    "readonly": true,

    "title": {
        "en": CONTROL_TITLE,    // English title of the control
        "ru": CONTROL_TITLE_RU, // Russian title of the control
        ...
    }

    // Enum titles for control's value. Could be set for type "value" and "text".
    // In case of type "value", each key in "enum" should be a stringified number, specified in either decimal or hexadecimal format.
    "enum": {
        "0": {
            "en": ENUM_TITLE,
            "ru": ENUM_TITLE_RU,
            ...
        },
        "1": {
            "en": ENUM_TITLE,
            "ru": ENUM_TITLE_RU,
            ...
        },
        ...
    }
}
```

`type`, `min`, `max`, `order`, `readonly` could be published as subtopics of `/devices/+/controls/+/meta` for backward compatibility with old conventions.

### Control Types

#### Switch
A control that toggles it's value when pressed by the user.
* Meta topic value: switch
* Possible values: 0 or 1

#### Alarm
A control that indicates whether an alarm is active.
* Meta topic value: alarm
* Possible values: 0 or 1

#### Push button
A stateless push button
* Meta topic value: pushbutton
* Possible values: 1
* Messages may lack retained flag

#### Range
A range slider that takes integer values between 0 and any other integer that is greater 1
* Meta topic value: range
* Possible values: min - max
* Default max: 255
* Default min: 0
Different values can be set by publishing an arbitrary integer that is in range from ```min``` to ```max```.

#### RGB color control
R/W control for color
* Meta topic value: rgb
* Possible values: "R;G;B", i.e. three semicolon-delimited numbers.
The numbers itself must be integers between 0 and 255.

#### Text
A read-only control that displays it's value as text.
* Meta topic value: text
* Possible values: Anything

#### Generic value type control

A control for a arbitrary value.

* Meta type value: value
* Possible values: float
Different values can be set by publishing an arbitrary float that is in range from ```min``` to ```max```.
Precision could be specified in ```precision``` property. The value is rounded to defined precision by a driver and it is also used by `wb-mqtt-homeui` during user input validation.

#### Specific value type controls

:warning: **WARNING**: These control types are deprecated. It is recommended to use `units` property instead.

| Type 	| meta/type	| units  	| value format  	|
|---	|---	|---	|---	|
| Temperature  	| temperature| °C  	| float  	|
| Relative humidity  	| rel_humidity| %, RH  	| float, 0 - 100  	|
| Atmospheric pressure  	| atmospheric_pressure | millibar (100 Pa)  	| float    	|
| Precipitation rate (rainfall rate) | rainfall | mm per hour | float |
| Wind speed |  wind_speed | m/s | float |
| Power |  power | watt | float |
| Power consumption |  power_consumption | kWh | float |
| Voltage |  voltage | volts | float |
| Water flow | water_flow | m^3 / hour | float |
| Water total consumption | water_consumption | m^3  | float |
| Resistance | resistance | Ohm  | float |
| Gas concentration | concentration | ppm  | float (unsigned) |
| Heat power | heat_power | Gcal / hour | float |
| Heat energy | heat_energy | Gcal | float |
| Current | current | A | float |
| Pressure | pressure | bar | float |
| Illuminance | lux | lx | float |
| Sound level | sound_level | dB | float |

#### Units

| Unit name | description |
|---        |---          |
| mm/h      | mm per hour, precipitation rate (rainfall rate) |
| m/s       | meter per second, speed |
| W         | watt, power |
| kWh       | kilowatt hour, power consumption |
| V         | voltage |
| mV        | voltage (millivolts) |
| m^3/h     | cubic meters per hour, flow |
| m^3       | cubic meters, volume |
| Gcal/h    | giga calories per hour, heat power |
| cal       | calories, energy |
| Gcal      | giga calories, energy |
| Ohm       | resistance |
| mOhm      | resistance (milliohms) |
| bar       | pressure |
| mbar      | pressure (100Pa) |
| s         | second |
| min       | minute |
| h         | hour |
| m         | meter |
| g         | gram |
| kg        | kilo gram |
| mol       | mole, amount of substance |
| cd        | candela, luminous intensity |
| %, RH     | relative humidity |
| deg C     | temperature |
| %         | percent |
| ppm       | parts per million |
| ppb       | parts per billion |
| A         | ampere, current |
| mA        | milliampere, current |
| deg       | degree, angle |
| rad       | radian, angle |

#### Errors

`/devices/+/controls/+/meta/error` topics can contain a combination of values:
- `r` - read from device error
- `w` - write to device error
- `p` - read period miss
