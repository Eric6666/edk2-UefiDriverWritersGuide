<!--- @file
  3.15.6 ConIn

  Copyright (c) 2012-2018, Intel Corporation. All rights reserved.<BR>

  Redistribution and use in source (original document form) and 'compiled'
  forms (converted to PDF, epub, HTML and other formats) with or without
  modification, are permitted provided that the following conditions are met:

  1) Redistributions of source code (original document form) must retain the
     above copyright notice, this list of conditions and the following
     disclaimer as the first lines of this file unmodified.

  2) Redistributions in compiled form (transformed to other DTDs, converted to
     PDF, epub, HTML and other formats) must reproduce the above copyright
     notice, this list of conditions and the following disclaimer in the
     documentation and/or other materials provided with the distribution.

  THIS DOCUMENTATION IS PROVIDED BY TIANOCORE PROJECT "AS IS" AND ANY EXPRESS OR
  IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
  MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO
  EVENT SHALL TIANOCORE PROJECT  BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
  SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
  PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS;
  OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
  WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR
  OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS DOCUMENTATION, EVEN IF
  ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

-->

### 3.15.6 ConIn

The platform connects the console devices using the device paths from the
_ConIn_, _ConOut_, and _ErrOut_ global UEFI variables. The _ConIn_ connection
process is discussed first.

  `ConIn = Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/ VenMsg(Vt100+)`

The UEFI connection process searches for the device in the handle database
having a device path that most closely matches the following.

  `Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/VenMsg(Vt100+)`

It finds handle 17 as the closest match. The portion of the device path that
did not match (`Uart(9600 N81)/VenMsg(Vt100+)`) is called the _remaining device
path._

  `17: PciIo DevPath (Acpi(HWP0002,0,PNP0A03)/Pci(1|1))`

UEFI calls `ConnectController()`, passing in handle 17 and the remaining device
path. The connection code constructs a list of all the drivers in the system
and calls each driver, passing handle 17 and the remaining device path into the
`Supported()` service. The only driver installed in the handle database that
returns `EFI_SUCCESS` for this device handle is handle 30:

  `30: Image(PciSerialDxe) DriverBinding ComponentName2 ComponentName`

After `ConnectController()` finds a driver that supports handle 17, it passes
device handle 17 and the remaining device path `Uart(9600 N81)/ VenMsg(Vt100+)`
into the serial driver's `Start()` service. The serial driver opens the PCI I/O
Protocol on handle 17 and create a new child handle. The following is installed
onto the new child handle:

* `EFI_SERIAL_IO_PROTOCOL` (defined in the Console Support chapter of the _UEFI
  Specification_)

* `EFI_DEVICE_PATH_PROTOCOL`

The device path for the child handle is generated by making a copy of the
device path from the parent and appending the serial device path node
`Uart(9600 N81)`. Handle 3B, shown below, is the new child handle.

  `3B: SerialIo DevPath (Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81))`

That first call to `ConnectController()` has now been completed, but the Device
Path Protocol on handle 3B does not completely match the _ConIn_ device path,
so the connection process is repeated. This time the closest match for 'Acpi(HWP0002,0)/Pci(1|1)/Uart(9600 N81)/VenMsg(Vt100+)' is the newly created device handle 3B. Now the remaining device path is 'VenMsg(Vt100+)'. The search for a driver that supports handle 3B finds the terminal driver, returning 'EFI_SUCCESS' from the 'Supported()' service.

  `31: Image(TerminalDxe) DriverBinding ComponentName2 ComponentName`

This driver's `Start()` service opens the `EFI_SERIAL IO_PROTOCOL`, creates a
new child handle, and installs the following:

* `EFI_SIMPLE_TEXT_INPUT_PROTOCOL`

* `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`

* `EFI_DEVICE_PATH_PROTOCOL`

The console protocols are defined in the Console Support chapter of the _UEFI
Specification_. The device path is generated by making a copy of the device
path from the parent and appending the terminal device path node
`VenMsg(Vt100+)`. VT100+ was chosen because that terminal type was specified in
the remaining device path that was passed into the `Start()` service. Handle
3C, shown below, is the new child handle.

  `3C: Txtin Txtout DevPath (Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/VenMsg(Vt100+))`

At this point, the process still has not completely matched the `ConIn` device path, so the connection process is repeated again. This time there is an exact match for `Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/VenMsg(Vt100+)` with the newly created child handle 3C. The search for a driver that supports this controller results in two driver handles that return EFI_SUCCESS to the Supported() service. The two driver handles are from the platform console management driver:

```
  32: Image(ConPlatformDxe) Driver Binding ComponentName2 ComponentName
  33: Driver Binding ComponentName2 ComponentName
```

Driver 32 installs a `ConOut` Tag GUID on the handle if the device path is
listed in the `ConOut` global UEFI variable. In this example, this case is true. Driver 32 also installs a _StdErr_ Tag GUID on the handle if the device path is listed in the `ErrOut` global UEFI variable. This case is also true in the following example. Therefore, handle 3C has two new protocols on it: `ConOut` and
`StdErr`.

```
  3C: TxtInEx Txtin Txtout ConOut StdErr DevPath (Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/ VenMsg(Vt100+))
```

Driver 33 installs a `ConIn` Tag GUID on the handle if the device path is
listed in the `ConIn` global UEFI variable (which it does because the
connection process started that way), so handle 3C has the `ConIn` protocol
attached.

```
  3C: TxtinEx Txtin Txtout ConIn ConOut StdErr DevPath (Acpi(HWP0002,0,PNP0A03)/Pci(1|1)/Uart(9600 N81)/VenMsg(Vt100+))
```

UEFI uses these three protocols (_ConIn_, _ConOut_, and _StdErr_) to mark
devices in the platform, which have been selected by the user as _ConIn_,
_ConOut_, and _StdErr_. These protocols are actually Tag GUIDs without any
services or data.

There are three other driver handles that return `EFI_SUCCESS` from the
`Supported()` service. These driver handles are from the console splitter
drivers for the _ConIn_, _ConOut_, and _StdErr_ devices in the system. There is
a fourth console splitter driver handle (which is not used on this handle) for
devices that support the Simple Pointer Protocol. The three driver handles are
listed below:

```
  34: Image(ConSplitterDxe) DriverBinding ComponentName3 ComponentName**
  35: DriverBinding ComponentName2 ComponentName**
  36: DriverBinding ComponentName2 ComponentName**
  37: DriverBinding ComponentName2 ComponentName**
```

Remember that when the console splitter driver was first loaded, it created
three virtual handles for the primary console input device, the primary console
output device, and the primary standard error device.

```
  38: TxtinEx TxtIn SimplePointer AbsolutePointer
  39: Txtout GraphicsOutput UgaDraw
  3A: Txtout
```

The console splitter driver's `Supported()` service for handle 34 examines the
handle 3C for a _ConIn_ Protocol. Having found it, it returns `EFI_SUCCESS`.
The `Start()` service then opens the _ConIn_ protocol on handle 3C such that
the primary console input device handle 38 becomes a child controller and starts aggregating the `SIMPLE_INPUT_EX_PROTOCOL` and `SIMPLE_INPUT_PROTOCOL` services. The same thing happens for handle 36 with _ConIn_, except that the

`SIMPLE_TEXT_OUTPUT_PROTOCOL` functionality on handle 3C is aggregated into the
`SIMPLE_TEXT_OUTPUT_PROTOCOL` on the primary console output handle 39.

Handle 37 with _StdErr_ also does the same thing; the
`SIMPLE_TEXT_OUTPUT_PROTOCOL` functionality on handle 3C is aggregated into the
`SIMPLE_TEXT_OUTPUT_PROTOCOL` on the primary standard error handle 3A.

The connection process has now been completed for _ConIn_ because the device
path that completely matched the _ConIn_ device path and all the
console-related services has been installed.