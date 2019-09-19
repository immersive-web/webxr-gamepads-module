# WebXR Gamepads Module - Level 1
This document explains the portion of the WebXR APIs for interacting with the buttons, triggers, thumbsticks, and touchpads of an `XRInputSource`. For context, it may be helpful to have first read the official [Core WebXR spec](https://www.w3.org/TR/webxr/) and the related explainers ([core explainer](https://github.com/immersive-web/webxr/blob/master/explainer.md) and [input explainer](https://github.com/immersive-web/webxr/blob/master/input-explainer.md)) found in the main [WebXR repo](https://github.com/immersive-web/webxr/).

## Concepts
Touchpad, thumbstick, trigger, and button data can be observed via an `XRInputSource`'s `gamepad` attribute. `gamepad` will be an instance of the [`Gamepad`](https://w3c.github.io/gamepad/#gamepad-interface) interface if the input source has buttons and axes to report, and `null` otherwise.

Examples of input sources that may expose their state this way include Oculus Touch, Vive wands, Oculus Go and Daydream controllers, or other similar devices. Input devices not directly associated with the XR device, such as the majority of traditional gamepads, and tracked devices without discreet inputs, such as optical hand tracking, must not be exposed using this interface. 

`Gamepad` instances reported in this way have several notable behavioral changes vs. the ones reported by `navigator.getGamepads()`:

  - `Gamepad` instances connected to an `XRInputSource` must not be included in the array returned by `navigator.getGamepads()`.
  - The `Gamepad`'s `id` attribute must be `""` (empty string).
  - The `Gamepad`'s `index` attribute must be `-1`.
  - The `Gamepad`'s `connected` attribute must be `true` unless the related `XRInputSource` is removed from the `inputSources` array or the related `XRSession` is ended.

The exact button and axes layout is given by the `XRInputSource`'s `profiles` attribute, which contains an array of strings that identify the button and axes layout or subsets of it, ordered from most specific to least specific.

```js
function onXRFrame(timestamp, frame) {
  let inputSource = primaryInputSource;

  // Check to see if the input source has gamepad data.
  if (inputSource && inputSource.gamepad) {
    let gamepad = inputSource.gamepad;
    
    // Use touchpad values for movement.
    if (gamepad.axes.length >= 2) {
      MoveUser(gamepad.axes[0], gamepad.axes[1]);
    }

    // If the first gamepad button is pressed, perform an action.
    if (gamepad.buttons.length >= 1 && gamepad.buttons[0].pressed) {
      EmitPaint();
    }
    
    // etc.
  }

  // Do the rest of typical frame processing...
}
```

If the application includes interactions that require user activation (such as starting media playback), the application can listen to the `XRInputSource`s `select` events, which fire for the primary button on the controller.

The UA may update the `gamepad` state at any point, but it must remain constant while running a batch of `XRSession` `requestAnimationFrame` callbacks or event callbacks which provide an `XRFrame`.

### XR gamepad mapping

The WebXR Device API also introduces a new standard controller layout indicated by the `mapping` value of `xr-standard`. (Additional mapping variants may be added in the future if necessary.) This defines a specific layout for the inputs most commonly found on XR controller devices today. The following table describes the buttons/axes and their associated physical inputs:

| Button     | `xr-standard` Mapping    |
| ---------- | -------------------------|
| buttons[0] | Primary button/trigger   |
| buttons[1] | Secondary button/trigger |
| buttons[2] | Touchpad press           |
| buttons[3] | Thumbstick press         |

| Axis    | `xr-standard` Mapping |
| ------- | ----------------------|
| axes[0] | Touchpad X            |
| axes[1] | Touchpad Y            |
| axes[2] | Thumbstick X          |
| axes[3] | Thumbstick Y          |

Additional device-specific inputs may be exposed after these reserved indices, but devices that lack one of the canonical inputs must still preserve their place in the array.

In order to make use of the `xr-standard` mapping, a device must meet **at least** the following criteria:

 - Is a `tracked-pointer` device. 
 - Has a trigger or similarly accessed button separate from any touchpads or thumbsticks

devices that do not meet that criteria may still expose `gamepad` data, but must not claim the `xr-standard` mapping. For example: The controls on the side of a Gear VR would not qualify for the `xr-standard` mapping because they represent a `gaze`-style input. Similarly, a Daydream controller would not qualify for the `xr-standard` mapping since it lacks a trigger.

### Exposing button/axis values with action maps

Some native APIs rely on what's commonly referred to as an "action mapping" system to handle controller input. In action map systems the developer creates a list of application-specific actions (such as "undo" or "jump") and suggested input bindings (like "left hand touchpad") that should trigger the related action. Such systems may allow users to re-bind the inputs associated with each action, and may not provide a mechanism for enumerating or monitoring the inputs outside of the action map.

When using an API that limits reading controller input to use of an action map, it is suggested that a mapping be created with one action per possible input, given the same name as the target input. For example, an similar mapping to the following may be used for each device:

| Button/Axis | Action name        | Sample binding              |
|-------------|--------------------|-----------------------------|
| button[0]   | "trigger"          | "[device]/trigger"          |
| button[1]   | "squeeze"             | "[device]/squeeze"             |
| button[2]   | "touchpad-click"   | "[device]/touchpad/click"   |
| button[3]   | "thumbstick-click" | "[device]/thumbstick/click" |
| axis[0]     | "touchpad-x"       | "[device]/touchpad/x"       |
| axis[1]     | "touchpad-y"       | "[device]/touchpad/y"       |
| axis[2]     | "thumbstick-x"     | "[device]/thumbstick/x"     |
| axis[3]     | "thumbstick-y"     | "[device]/thumbstick/y"     |

If the API does not provided a way to enumerate the available input devices, the UA should provide bindings for the left and right hand instead of a specific device and expose a `Gamepad` for any hand that has at least one non-`null` input.

The UA must not make any attempt to circumvent user remapping of the inputs.

### Generic Profiles

All `XRInputSource` objects report a `profiles` array describing the device being used with varying levels of detail, ranging from exactly identifying the device to only giving a broad description of it's shape and capabilities. It is highly recommended that `XRInputSource`s which expose `Gamepad` data ensure that the last profile in the array be from the list of well-known "generic" profiles, given below.

 - **"button-controller":** A controller with at least one button/trigger but no touchpad or thumbstick. Controllers with this profile must use the `xr-standard` Gamepad mapping.
 - **"touchpad-controller"** A controller with a touchpad, but no thumbstick. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.
 - **"thumbstick-controller"** A controller with a thumbstick, but no touchpad. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.
 - **"touchpad-thumbstick-controller"** A controller with both a touchpad and a thumbstick. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.

More generic profiles may be added to this list over time as new common form factors are observed.

## Appendix A: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](https://github.com/immersive-web/webxr/blob/master/explainer.md).

```webidl
//
// Input
//

partial interface XRInputSource {
  readonly attribute Gamepad? gamepad;
};
```
