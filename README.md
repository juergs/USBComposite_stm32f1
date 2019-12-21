# USB Composite library for Roger's Melbourne's STM32F1 core: https://github.com/rogerclarkmelbourne/Arduino_STM32/

## Protocols supported

- standard USB HID, with many built-in profiles, and customizable with more

- MIDI over USB

- XBox360 controller (only controller-to-host is currently supported)

- Mass storage

## Basic concepts

Start with:
```
#include <USBComposite.h>
```

The library defines one central singleton object:

```
extern USBCompositeDevice USBComposite;
```

This controls USB device identification as well as registers the plugins that are connected to it.

Plugin classes included in the library: 

```
USBHID
USBMIDI
USBXBox360
USBMassStorage
USBCompositeSerial
```

To use them, you need to create instances of them. Currently, only one instance of each plugin class
can be created.

If you want to make a simple (non-composite) USB device, you can create an instance of the plugin class
and call the `begin()` method, and it will take care of registering itself with `USBComposite` and starting up
`USBComposite`. If you want to make a composite USB device, however,
you need to control the device with `USBComposite`:

```
USBComposite.clear(); // clear any plugins previously registered
plugin1.registerComponent(); 
plugin2.registerComponent();
USBComposite.begin();
```

Of course, you may need to do some further configuring of the plugins (e.g., if plugin1 is USBHID, then
you may want to do `USBHID.setReportDescriptor(HID_KEYBOARD)`) or of the `USBComposite` device
before the `USBComposite.begin()` call.

Finally, there are a number of classes that implement particular protocols for the `USBHID` class plugin.
These are:
```
HIDMouse
HIDKeyboard
HIDJoystick
HIDAbsMouse
HIDConsumer
HIDRaw<txSize,rxSize>
```
And you can customize with more. Moreover, the `USBHID` plugin itself allows for compositing
multiple HID profiles, e.g., Mouse / Keyboard / three joysticks. Each of these has at least
one required parameter, which is an instance of `USBHID`.

Not all combinations will fit within the constraints of the STM32F1 USB system, and not all
combinations will be supported by all operating systems.

I recommend calling `USBComposite.setDeviceId(device)` with a different device number for each combination
of plugins and profiles to prevent problems with cached configurations on the host computer.

## Uploading with STM32duino bootloader

Normally, the STM32duino bootloader upload method in the Roger Melbourne STM32F1 core sends a command 
to reset the board via the USB serial port, and thereby put it in bootloader mode, just prior to uploading. 
If you have installed a sketch that includes a USB serial port in the composite device, this should still
work. But if the sketch you've installed doesn't include a USB serial port, then you need to manually activate 
the bootloader mode next time you want to upload a sketch.

The bootloader mode is active for a short period after the board powers up or resets. So just initiate
the upload in the Arduino IDE as usual, but when "Searching for DFU device [1EAF:0003]" is displayed,
hit the reset button (or if the device is USB-powered, keep it unplugged from USB and plug it in when you 
get this message).

## Simple USB device configuration

A simple USB device uses a single plugin. You need to create an instance of the plugin class,
call any setup methods for the plugin and the `begin()` method for the plugin. For instance, 
to inject keyboard data, you should do:

```
USBHID HID; // create instance of USBHID plugin
HIDKeyboard Keyboard(HID); // create a profile

HID.begin(HID_KEYBOARD);
```

and then call `Keyboard.print("TextToInject")` to inject keyboard data. Some plugin configurations
may require further initialization code or further code that needs to be called inside the Arduino
`loop()` function.

See the `BootKeyboard`, `midiout` and `x360` example code for this procedure.

(Additionally, the `USBHID` plugin has a convenience `begin()` method that lets you include an
instance of a `USBCompositeSerial` plugin class, and that creates a composite HID-Serial device.)

However, if you want a USB device using more than one plugin, then you will NOT call the plugin's
`begin()` method.

## Memory limitations

There are 320 bytes of hardware buffer memory available after endpoint 0 is taken into account. The following 
are the default buffer memory needs of the current components:

 * USB Serial: 144 bytes
 
 * USB HID: 64 bytes
 
 * USB Mass Storage: 128 bytes
 
 * USB MIDI: 128 bytes
 
 * XBox360 Controller: 64 bytes
 
This places a limit on what combinations can be used together. For instance, HID+Mass storage+MIDI should be theoretically 
OK (320 bytes), but Serial+HID+Mass storage (336 bytes) will fail with default settings (and return false from 
USBComposite.begin()) due to lack of memory.

However, USB Serial, USB HID and USB MIDI allow you to decrease buffer sizes (and allow for more complex composite devices)
by calling:
```
USBHID HID;
USBCompositeSerial CompositeSerial;
USBMIDI MIDI;

CompositeSerial.setRXPacketSize(size);
CompositeSerial.setTXPacketSize(size);
HID.setTXPacketSize(size); 
MIDI.setRXPacketSize(size);
MIDI.setTXPacketSize(size);
```
The maximum and default packet size is 64. Smaller packet sizes have not been thoroughly tested and may slow things down. In 
particular, for HID you should make sure your packet size is sufficient for your largest HID report. The CompositeSerial 
device also has a control channel whose 16 byte packet size is not adjustable.

Note that in the above, RX and TX are from the point of view of the MCU, not the host (i.e., RX corresponds to USB Out and TX
to USB In).

## Endpoint limitations

There is one bidirectional endpoint 0 that all endpoints share, and the hardware allows for seven more in each direction. 
Each plugin contributes the following count towards the maximum of seven:

* USB Serial: 2 (= 2 TX, 1 RX)

* USB HID: 1 (= 1 TX)

* USB Mass Storage: 1 (= 1 TX, 1 RX)

* USB MIDI: 1 (= 1 TX, 1 RX)

* XBox360 Controller: 1 (= 1 TX, 1 RX)

* USB Audio: 1 (= 1 TX or 1 RX depending on mode)

When combining plugins, make sure the contribution does not exceed 7, though with the above list of plugins you can't exceed that.
