## Wiren Board MQTT Conventions

### 1. Basic Concepts

The basic abstractions are *devices* and their *controls*.

#### 1.1. Root Path

All devices must be located at the root level of the MQTT broker under the `/devices/...` topic (e.g., `/devices/<!new_device_name!>/...`).

Each *device* has some *controls* assigned to it, i.e. parameters that can be controlled or monitored.
*Devices* and *controls* are identified by names, which are generally arbitrary strings.

#### 1.2. Naming Conventions (2024+)

Starting from 2024, WirenBoard has established the following rules for naming MQTT topics for new devices and their controls:
- The topic name cannot include punctuation, brackets, or special characters such as %$#& etc.
- For a new device, the topic name should not contain more than four words and numbers. Write each word in lowercase and separate them with underscores.
- For a new version of an existing device, make the topics compatible with the old pattern (marked as deprecated). The names of new topics should be styled identically to the existing topics.

Examples:
- Good: `/devices/room_light/meta`
- Bad: `/devices/Room-Light#1/meta`
- Old: `/devices/RoomLight/meta` - not recommended for new topics

#### 1.3. Device and Controls Hierarchy example
For example, some room lighting control *device* with one input (for wall switch) and one output (for controlling the lamp) *controls* is represented with MQTT topics as following:

* `/devices/room_light/meta` - JSON with all meta information about *device*
* `/devices/room_light/meta/error` - device-level error state, non-null means there was an error (usable as Last Will and Testament)
* `/devices/room_light/controls/lamp` - contains current lamp state, '0' = off, '1' = on
* `/devices/room_light/controls/lamp/on` - send a message with this topic and payload of '0'/'1' to turn lamp off or on
* `/devices/room_light/controls/lamp/meta` - JSON with all meta information about control
* `/devices/room_light/controls/switch` - contains current wall switch state
* `/devices/room_light/controls/switch/meta` - JSON with all meta information about control
* `/devices/room_light/controls/switch/meta/error` - non-null value means there was an error reading or writing the control. In this case  `/devices/room_light/controls/switch` contains last known good value.

Each *device* usually represents the single physical device or one of the integrated peripheral of a complex physical device, although there are some boundary cases where the distinction is not clear. The small and not-so-complex real-world devices (say, wireless weather sensor) are ought to be represented by a single *device* in the MQTT hierarchy. 
Each *device* must be handled by a single driver or publisher, though it's not enforced in any way.

The *Conventions* are based on [HomA MQTT Conventions](https://github.com/binarybucks/homA/wiki/Conventions). The main changes are: no configuration is stored in MQTT (as MQTT is not so good as a database) and the *control* types system is more developed and complicated.

#### 1.4. Value and Command Topics

Every *control* has a value topic:

* `/devices/<device>/controls/<control>` — the **current** value of the control, published by the driver.

A **writable** control has, in addition, a command topic:

* `/devices/<device>/controls/<control>/on` — the **desired** value of the control, published by a client (web interface, rules engine, third-party software).

A read-only control has no `/on` topic: nothing publishes to it and no driver subscribes to it. Whether a control is writable is defined by `readonly`, see [2.2](#22-controlss-meta-topic).

`/on` works the same way for **every** writable control regardless of its type: there is no control type that is written to in some other way.

A driver applies the requested value to the physical device and only then publishes the resulting value to the control's topic. Consequently:

* the two topics may hold different values while the change is in progress;
* the driver may publish a value that differs from the requested one, for example when the device clamps or rounds it, or refuses the write;
* a client must not consider a value applied until it has been published to the control's topic.

The payload format of `/on` is the same as that of the control's topic and is defined by the control's type, see [3](#3-control-types).

Unlike the value topic, `/on` is **not** retained — it carries a command, not a state, see [1.5](#15-retained-flag).

```
/devices/wb-mrm2_130/controls/Relay 1/on   1   # a client asks to turn the relay on
/devices/wb-mrm2_130/controls/Relay 1      1   # the driver reports the relay is on
```

#### 1.5. Retained Flag

| Topic | `retained` |
|---                                                          |---  |
| `/devices/+/meta` and `/devices/+/meta/+`                   | yes |
| `/devices/+/controls/+/meta` and `/devices/+/controls/+/meta/+` | yes |
| `/devices/+/controls/+`                                     | yes, except `pushbutton` |
| `/devices/+/controls/+/on`                                  | **no** |

Values and metadata are published with the `retained` flag so that a client connecting later immediately receives the current state of the system instead of waiting for the next update.

Messages published to `/on` must **not** be retained. A retained command would be redelivered to the driver on every reconnect and would re-apply an outdated value.

`pushbutton` is stateless, so its value messages are published without the `retained` flag.

The Conventions do not prescribe a QoS level, and existing drivers use different ones, so a client must not expect any particular QoS.

#### 1.6. Removing Devices and Controls

Because the topics are retained, a device that is no longer published remains visible to clients until its topics are cleared. To remove a device or a control, publish an empty payload with the `retained` flag to each of its topics. The `mqtt-delete-retained` utility does that for a topic mask:

```
mqtt-delete-retained '/devices/wb-mrm2_130/#'
```

An empty payload in `/devices/+/meta` removes the device, an empty payload in `/devices/+/controls/+/meta` removes the control.

### 2. Metadata Publishing

The metadata is published exclusively in the `*/meta` topics and their subtopics.
Metadata messages are published on device startup with `retained` flag set.

#### 2.1. Device's `/meta` topic

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


#### 2.2. Controls's `/meta` topic

The topic contains all meta information in one JSON

```jsonc
{
    // Control's type
    "type": "value",

    // Units. ASCII string. Meaningful for the numeric types "value" and "range".
    // No units by default
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

    // The control has no /on topic and cannot be written to.
    // There is no single default: if the key is absent, the default of the control's
    // type is used, see section 3. Publishing "readonly": false is therefore not the
    // same as omitting the key — for "value" and "text", which are read-only
    // by default, it is what makes the control writable.
    // An empty payload in the /meta/readonly subtopic resets the control
    // to the default of its type.
    "readonly": true,

    // Current error state of the control, see section 4.
    // Clients accept the error state here as well, but the usual place for it
    // is the /meta/error subtopic, because the error state changes at runtime
    // while the rest of the metadata does not
    "error": "r",

    // The control is for internal use only. Do not show it in web-interface. Default value is false
    "hidden": true,

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

#### 2.3. Legacy `/meta/<key>` Subtopics

Before the JSON `/meta` topic was introduced, each metadata key was published in its own subtopic. Clients still understand these subtopics, and many drivers still publish them alongside the JSON, so a client should handle both forms. Which of them a particular driver publishes varies — the table below lists what a client accepts, not what a driver is required to send.

A subtopic contains a **bare scalar value, not JSON**: the payload of `/devices/foo/controls/bar/meta/type` is `range`, not `"range"`.

| Subtopic                               | Key in JSON `/meta`     | Payload |
|---                                     |---                      |---      |
| `/devices/+/meta/driver`               | `driver`                | string  |
| `/devices/+/meta/name`                 | device's `title.en`     | string  |
| `/devices/+/controls/+/meta/type`      | `type`                  | control type name, e.g. `range` |
| `/devices/+/controls/+/meta/name`      | control's `title.en`    | string  |
| `/devices/+/controls/+/meta/units`     | `units`                 | string  |
| `/devices/+/controls/+/meta/min`       | `min`                   | number  |
| `/devices/+/controls/+/meta/max`       | `max`                   | number  |
| `/devices/+/controls/+/meta/precision` | `precision`             | number  |
| `/devices/+/controls/+/meta/order`     | `order`                 | integer |
| `/devices/+/controls/+/meta/readonly`  | `readonly`              | `1`, `0` or empty |
| `/devices/+/controls/+/meta/error`     | `error`                 | see section 4 |

`/meta/error` is listed here for completeness, but it is not a legacy topic: it is the usual place for the error state, see section 4.

There is no subtopic for `title` with several languages, `enum` and `hidden` — these keys exist only in the JSON `/meta` topic.

A new driver should publish the JSON `/meta` topic. Publishing both forms is allowed, and drivers that must stay compatible with old clients do publish both; in that case the driver is responsible for keeping them consistent, since for a client the value that arrives last wins.

`/devices/+/controls/+/meta/writable` is deprecated and ignored — publish `readonly` = `0` instead.

### 3. Control Types

The control's type defines the payload format of the control's topic — and of its `/on` topic, if the control is writable. It also determines how the control is presented in the web interface and whether it is writable when `readonly` is not set.

| Type          | Payload         | Web interface representation |
|---            |---              |---                           |
| `switch`      | `0` or `1`      | toggle                       |
| `alarm`       | `0` or `1`      | alert indicator              |
| `pushbutton`  | `1`             | button                       |
| `range`       | number          | slider                       |
| `rgb`         | `R;G;B`         | color picker                 |
| `text`        | any string      | text, input field if writable |
| `unixtime`    | integer         | date and time picker         |
| `w1-id`       | 64-bit unsigned integer | formatted identifier |
| `value`       | float           | number, input field if writable |

If `readonly` is not set, the control is writable for `switch`, `pushbutton`, `range`, `rgb` and `unixtime`, and read-only for `alarm`, `text`, `w1-id`, `value` and the deprecated type-specific value controls.

A control with `enum` in its metadata is shown as a drop-down list instead of the representation above, see the "Enum" section below.
The deprecated type-specific value controls listed at the end of this section behave like `value`.

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
* The control has no state, so its messages are published without the `retained` flag, see [1.5](#15-retained-flag)

#### Range
A numeric control shown as a slider.
* Meta topic value: range
* Possible values: numbers from ```min``` to ```max```

```min```, ```max``` and ```precision``` are taken from the control's metadata, see [2.2](#22-controlss-meta-topic). If they are omitted, the defaults documented there apply: ```min``` is 0 and ```max``` is 10^9. ```precision``` is the step of the slider; if it is omitted the step is 1, so the control accepts integers only.

A driver should publish ```min``` and ```max``` explicitly rather than rely on the defaults: a slider spanning the whole default interval is useless in the user interface.

Different values can be set by publishing to ```/on``` a number that is in range from ```min``` to ```max``` and is a multiple of ```precision```.

The difference from `value` is presentation, not the payload: `range` is rendered as a slider and `value` as a number with an input field. A slider needs a bounded interval, so use `range` when the control has a meaningful ```min``` and ```max``` (dimming level, fan speed) and `value` when it does not (a measured temperature or power).

#### RGB color control
R/W control for color
* Meta topic value: rgb
* Possible values: "R;G;B", i.e. three semicolon-delimited numbers.
The numbers itself must be integers between 0 and 255.

#### Text
A control that displays a value as text.
* Meta topic value: text
* Possible values: Anything

Like `value`, the control is read-only unless ```readonly``` is explicitly set to `false`, see [2.2](#22-controlss-meta-topic). A writable control is shown as an input field, or as a drop-down list if ```enum``` is set.

#### Unix time
A control type for unix time.
* Meta topic value: unixtime
* Possible values: integer unix time

#### 1-Wire Device Identifier
A control type for Dallas 1-Wire device identifiers (addresses). 
* Meta topic value: w1-id
* Possible values: 64-bit unsigned integer

#### Generic value type control

A control for a arbitrary value.

* Meta type value: value
* Possible values: float

The control is read-only unless ```readonly``` is explicitly set to `false`, see [2.2](#22-controlss-meta-topic). A writable control is shown as an input field.

Different values can be set by publishing an arbitrary float that is in range from ```min``` to ```max```.
Precision could be specified in ```precision``` property. The value is rounded to defined precision by a driver and it is also used by the web interface during user input validation. Unlike `range`, `value` has no default ```min``` and ```max```: if they are omitted, the value is not bounded.

#### Enum

The ```enum``` metadata property turns a `value` or `text` control into a control with a fixed set of allowed values. It is not a control type of its own — ```type``` stays `value` or `text`.

A control with ```enum``` is presented in the web interface as a **drop-down list** of the titles from ```enum```, in the language chosen by the user. If a title for that language is missing, the English one is used; if there is no English one either, the raw value is shown.

A read-only control with ```enum``` shows the title corresponding to its current value instead of the value itself.

The topics still carry the raw value, not the title: a client publishes `1` to ```/on```, not `On`.

A value that is not listed in ```enum``` is shown as is. Drivers should not publish such values.

```jsonc
{
    "type": "value",
    "readonly": false,
    "enum": {
        "0": { "en": "Off",  "ru": "Выключено" },
        "1": { "en": "On",   "ru": "Включено" },
        "2": { "en": "Auto", "ru": "Автоматически" }
    }
}
```

#### Specific value type controls

:warning: **WARNING**: These control types are deprecated. It is recommended to use `units` property instead.

| Type                               | meta/type            | units       | value format |
|---                                 |---                   |---          |---           |
| Temperature                        | temperature          | deg C       | float |
| Relative humidity                  | rel_humidity         | %, RH       | float, 0 - 100 |
| Atmospheric pressure               | atmospheric_pressure | mbar        | float |
| Precipitation rate (rainfall rate) | rainfall             | mm/h        | float |
| Wind speed                         | wind_speed           | m/s         | float |
| Power                              | power                | W           | float |
| Power consumption                  | power_consumption    | kWh         | float |
| Voltage                            | voltage              | V           | float |
| Water flow                         | water_flow           | m^3/h       | float |
| Water total consumption            | water_consumption    | m^3         | float |
| Resistance                         | resistance           | Ohm         | float |
| Gas concentration                  | concentration        | ppm         | float (unsigned) |
| Heat power                         | heat_power           | Gcal/h      | float |
| Heat energy                        | heat_energy          | Gcal        | float |
| Current                            | current              | A           | float |
| Pressure                           | pressure             | bar         | float |
| Illuminance                        | lux                  | lx          | float |
| Sound level                        | sound_level          | dB          | float |

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
| g/m^3     | gram per cubic meter, gas concentration |
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
| lx        | illuminance |
| dB        | sound level |
| Hz        | hertz, frequency |
| rpm       | revolutions per minute |
| Pa        | pressure |
| var       | volt-ampere reactive, reactive power |
| VA        | volt-ampere, apparent power |
| kvarh     | kilovolt-ampere reactive hour, reactive energy |

### 4. Errors

The error state of a control is published in `/devices/+/controls/+/meta/error`. It may also be carried by the `error` key of the control's JSON `/meta`, which clients accept, but drivers normally use the subtopic: unlike the rest of the metadata, the error state changes at runtime, and the subtopic lets a driver update it without republishing the whole JSON. A driver that uses both must keep them consistent.

The error state of a whole device is published in `/devices/+/meta/error`; it is meant to be used as the Last Will and Testament of the driver's MQTT connection.

The payload is a combination of the following flags, in any order, without separators:
- `r` - failed to read from device or a device reports an error
- `w` - write to device error
- `p` - read period miss

An empty payload means there is no error. The topic is retained, like any other metadata, so a driver must publish an empty payload to clear an error rather than simply stop publishing.

An error does not change the control's value topic, and the control is neither removed nor emptied. The control's topic keeps the last known good value, and clients must keep treating it as the last known value rather than as the current one. The web interface keeps showing that value and marks the control as faulty: `r` and `w` are shown as an error highlight, `p` as a separate poll warning.

The order of publishing is significant:
- when setting `r`, the control's topic must already contain the last known good value;
- after a successful read, the `r` flag is removed first and the new value is published after that, so that a client never sees a fresh value together with a stale read error;
- the `w` flag is removed only after a successful write, regardless of whether reading succeeds.
