# ![](icon.svg) Unified Controller Input Specification

- [The Problem](#the-problem)
- [Summary](#summary)
- [Input Capabilities](#input-capabilities)
- [Input Value](#input-value)
   * [Binary Value](#binary-value)
   * [Uint8 Value](#uint8-value)
   * [Vector2 Value](#vector2-value)
   * [Vector3 Value](#vector3-value)
   * [Touch Value](#touch-value)
- [Input Capability Info](#input-capability-info)
- [Output Capabilities](#output-capabilities)
- [Output Capability Info](#output-capability-info)
- [HID Reports](#hid-reports)
   * [InputCapabilityReport](#inputcapabilityreport)
   * [InputDataReport](#inputdatareport)
      + [Example](#example)
- [Feature Reports](#feature-reports)
- [FAQ](#faq)
- [Attributions](#attributions)

## The Problem

The current state of handheld controller support on Linux is a bit of a mess.
Many handheld gaming PCs such as the Legion Go, ROG Ally, and Ayaneo were designed
to only work on Windows coupled with vendor-specific software. As a consequence,
this has made supporting these devices on Linux difficult; providing a wide range
of different input implementations each with their own quirks and workarounds to
enable all the hardware features of these devices. In order to expose the features
of these devices to games, typically a separate service is required to get around
these quirks and emulate a well known controller such as the Steam Deck Controller
or DualSense Controller. By doing this, though, we leave behind any hardware
features these devices have that games won't be able to utilize.

This repository contains information and documentation to address these issues:

The **Unified Controller Input Specification**

The Unified Controller Input Spec is meant to define a standard HID
interface that a program (such as [InputPlumber](https://github.com/ShadowBlip/InputPlumber/) or [HHD](https://github.com/hhd-dev/hhd)) can emulate via [uhid](https://kernel.org/doc/Documentation/hid/uhid.txt) or equivalent
to provide a unified input device for handhelds that is hardware agnostic for
applications to consume. In the future, this standard may also provide an interface
for hardware manufacturers to target directly in hardware to work in Linux without any
additional software.

Together with input library adoption (such as from [SDL](https://github.com/libsdl-org/SDL)) and a kernel driver,
applications can take full advantage of the hardware capabilities
of their input devices through a single unified HID interface.


## Summary

The Unified Controller works by providing two main HID reports through
the [HIDRAW subsystem](https://www.kernel.org/doc/html/latest/hid/hidraw.html):

* `InputCapabilityReport`: describes what input capabilities the device implements and how to decode the `InputDataReport`
* `InputDataReport`: contains the actual input data values

When an application wants to request the capabilities of a Unified Controller,
it can send a `GetInputCapabilities` feature report to the device, and the device will
respond with an `InputCapabilityReport` that describes the input capabilities of the
device and how it can decode the values in the `InputDataReport`. The device may also
send an `InputCapabilityReport` whenever it detects that its capabilities have changed.

The `InputCapabilityReport` also contains major and minor version information,
so the spec can be expanded to include new capabilities and maintain backwards
compatibility.

Conversely with output events (such as Force Feedback, RGB LEDs, etc.), the controller
also provides an HID report for output capabilities:

* `OutputCapabilityReport`: describes what output capabilities the device implements

## Input Capabilities

An input capability defines what type of input a physical device is capable
of handling. Each capability describes either a specific function (e.g. "Guide"
button that opens up a main menu, "Mute" button to mute audio, etc.) or an input
that exists in a specific physical location (e.g. "Left Stick", "Right Stick", 
"Center Touchpad", "Left Touchpad", "Right Touchpad").

Input capabilities are defined as a 16-bit enumeration of all possible capabilities, 
providing a maximum of 65535 possible input capabilities.

```rust
enum InputCapability {
  GAMEPAD_BUTTON_SOUTH     = 0,
  GAMEPAD_BUTTON_NORTH     = 1,
  GAMEPAD_BUTTON_EAST      = 2,
  GAMEPAD_BUTTON_WEST      = 3,
  GAMEPAD_BUTTON_GUIDE     = 4,
  GAMEPAD_AXIS_LEFT_STICK  = 5,
  GAMEPAD_AXIS_RIGHT_STICK = 6,
  GAMEPAD_GYRO_LEFT        = 7,
  ...
}
```

The full enumeration can be found [here](input_capability.md).

**NOTE: We could either use one flat enum of all capabilities, or multiple enums
based on input value type.**

E.g.

```rust
enum BinaryCapability {
  GAMEPAD_BUTTON_GUIDE = 0,
}
```

```rust
enum Vector2Capability {
  GAMEPAD_AXIS_LEFT_STICK = 0,
}
```

## Input Value

Input values are defined as an 8-bit enumeration of possible input values. This is used
to help decode the data in the `InputDataReport` by giving the type of data that
the value should be interpreted as, as well as its size in bits/bytes.

```rust
enum ValueType {
  /// Binary values take up 1 bit in the InputDataReport
  BINARY  = 0, // {bool}
  /// Uint8 values take up 1 byte in the InputDataReport
  UINT8   = 1, // {u8}
  /// Uint16 values take up 2 bytes in the InputDataReport
  UINT16  = 1, // {u16}
  /// Vector2 values take up 4 bytes in the InputDataReport
  VECTOR2 = 2, // {i16, i16}
  /// Vector3 values take up 6 bytes in the InputDataReport
  VECTOR3 = 3, // {i16, i16, i16}
  /// Touch values take up 6 bytes in the InputDataReport
  TOUCH   = 4, // {u8, bool, u8, u16, u16}
}
```

### Binary Value

Binary values can be decoded as:

```rust
struct BinaryValue {
    value: bool,
}
```

### Uint8 Value

Uint8 values can be decoded as:

```rust
struct Uint8Value {
    value: u8,
}
```

### Vector2 Value

Vector2 values can be decoded as:

```rust
struct Vector2Value {
    x: i16,
    y: i16
}
```

### Vector3 Value

Vector3 values can be decoded as:

```rust
struct Vector3Value {
    x: i16,
    y: i16,
    z: i16,
}
```

### Touch Value

Touch values can be decoded as:

```rust
struct TouchValue {
    finger_index: u8,
    is_touching: bool,
    pressure: u8,
    x: u16,
    y: u16,
}
```

## Input Capability Info

Capability Info describes the type of capability, and how to decode the value
of that capability in the `InputDataReport`.

```rust
/// The InputCapabilityInfo describes a single input capability that a device
/// implements and can be used to decode the values in the InputDataReport.
/// It is 5 bytes in size in the InputCapabilityReport.
struct InputCapabilityInfo {
    /// The capability
    capability: InputCapability, // u16
    /// The type of value this capability emits
    value_type: ValueType, // u8
    /// The bit offset in the input data report data to read the value for this capability.
    bit_offset: u16,
}
```

## Output Capabilities

An output capability defines what type of output a physical device is capable of
handling (such as force feedback, LED control, etc.).

**TODO**

## Output Capability Info

**TODO**

## HID Reports

### InputCapabilityReport

The `InputCapabilityReport` describes what capabilities are available for the
input device and how to decode the `InputDataReport`:

```rust
/// The InputCapabilityReport defines what the source gamepad is capable of emitting
struct InputCapabilityReport {
    /// The ID of the report
    report_id: u8, // always 0x01
    /// Major version indicates whether or not compatibility-breaking changes
    /// have occurred.
    major_ver: u8,
    /// The minor version indicates what capabilities are available. This version
    /// will be incremented when new capabilities are added to the spec.
    minor_ver: u8,
    /// The number of capabilities the device supports
    input_capabilities_count: u8, // maybe u16?
    /// List of input capability information
    input_capabilities: InputCapabilityInfo[],
}
```

This report will be sent in response to a `GetInputCapabilities` feature report
request from the application or if the input capabilities of the device have
changed.


### InputDataReport

The `InputDataReport` contains the actual input data for the controller and can
be decoded based on what input capabilities are in the `InputCapabilityReport`.
This report will be sent at every polling interval:

```rust
struct InputDataReport {
    /// The ID of the report
    report_id: u8, // always 0x02
    /// The state version field will increment any time the input report has changed.
    /// If the state version has not changed since the last frame, the report does
    /// not need to be processed.
    state_version: u8,
    /// Contains the input data that can be decoded based on the input capabilities
    /// defined in the InputCapabilityReport.
    data: byte[],
}
```

#### Example

Let's say we have a controller with only a single button. An unpacked capability report would
look like this:

```rust
InputCapabilityReport {
    report_id: 0x01,
    major_ver: 1,
    minor_ver: 1,
    input_capabilities_count: 1,
    input_capabilities: [
        InputCapabilityInfo {
            capability: InputCapability::GAMEPAD_BUTTON_GUIDE,
            value_type: InputValue::BINARY,
            bit_offset: 0,
        }
    ],
}
```

Using the information from the `InputCapabilityReport`, the `InputDataReport`
bytes would be sent like this:

```
| byte 0 | 00000010 | (report_id)
| byte 1 | 00000001 | (state_version)
| byte 2 | 10000000 | (data)
```

This data, unpacked according to the capability report, would look like this:

```rust
InputDataReport {
    report_id: 0x02,
    state_version: 1,
    BinaryValue {
      value: true,
    },
}
```

Here is a more complex example with 4 capabilities:

```rust
InputCapabilityReport {
    report_id: 0x01,
    major_ver: 1,
    minor_ver: 1,
    input_capabilities_count: 3,
    input_capabilities: [
        InputCapabilityInfo {
            capability: InputCapability::GAMEPAD_BUTTON_GUIDE,
            value_type: InputValue::BINARY,
            bit_offset: 0,
        },
        InputCapabilityInfo {
            capability: InputCapability::GAMEPAD_BUTTON_SOUTH,
            value_type: InputValue::BINARY,
            bit_offset: 1,
        },
        InputCapabilityInfo {
            capability: InputCapability::GAMEPAD_AXIS_LEFT_STICK,
            value_type: InputValue::VECTOR2,
            bit_offset: 8,
        },
        InputCapabilityInfo {
            capability: InputCapability::GAMEPAD_GYRO_LEFT,
            value_type: InputValue::VECTOR3,
            bit_offset: 32,
        },
    ],
}
```

This example would produce an `InputDataReport` with these bytes:

```
| byte 0  | 00000010 | (report_id)
| byte 1  | 00000011 | (state_version)
| byte 2  | 01000000 | (guide button and south button data)
| byte 3  | 00000000 | (left stick data)
| byte 4  | 00000000 |
| byte 5  | 00000000 |
| byte 6  | 00000000 |
| byte 7  | 00000000 | (gyro data)
| byte 8  | 00000000 |
| byte 9  | 00000000 |
| byte 10 | 00000000 |
| byte 11 | 00000000 |
| byte 12 | 00000000 |
```

Unpacked would be decoded as:

```rust
InputDataReport {
    report_id: 0x02,
    state_version: 3,
    BinaryValue {
      value: false,
    },
    BinaryValue {
      value: true,
    },
    Vector2Value {
      x: 0,
      y: 0,
    },
    Vector3Value {
      x: 0,
      y: 0,
      z: 0,
    },
}
```

## Feature Reports

Applications can send several different feature reports to the device for
different purposes:

- `GetInputCapabilities`: get the input capabilities of the controller
- `GetOutputCapabilities`: get the output capabilities of the controller
- `GetName`: gets the name of the controller
- `GetVendorId`: get the vendor id of the controller
- `GetProductId`: get the product id of the controller
- `GetSerial`: get the serial number of the controller
- `SetUnifiedMode`: sets the controller to be in unified input mode

The report IDs of each of these feature reports is defined as an 8-bit enumeration:

```rust
enum FeatureReportType {
    GET_INPUT_CAPABILITIES = 0x01,
    GET_OUTPUT_CAPABILITIES = 0x02,
    GET_NAME = 0x03,
    GET_VENDOR_ID = 0x04,
    GET_PRODUCT_ID = 0x05,
    GET_SERIAL = 0x06,
    SET_UNIFIED_MODE = 0x80,
}
```

## FAQ

- How is this system any different than just using HID report descriptors?

Report descriptors can describe the number of buttons, joysticks, etc., but
may not include correct usage info. E.g. what is button 1 used for? These require
specialized drivers or a mapping database to correctly map these inputs to evdev events.

## Attributions

Artwork

* [Gamepad Icon](https://kenney.nl/assets/input-prompts) by [Kenny](https://kenney.nl/) is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
* [Atom Icon](https://icon-sets.iconify.design/fontisto/atom/) by [Kenan Gündoğan](https://github.com/kenangundogan/fontisto) is licensed under [MIT](https://github.com/kenangundogan/fontisto/blob/master/LICENSE)

Contributors

* [William Edwards](https://github.com/shadowapex)
* [Derek Clark](https://github.com/pastaq/)
* [Denis Benato](https://github.com/NeroReflex/)
