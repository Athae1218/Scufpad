# Scufpad — personal fixes fork

This repository is my fork of [Scufpad](https://github.com/ChaseDRedmon/Scufpad),
originally created by **Chase Redmon**.
I forked it to address issues I experienced while using my Scuf Envision Pro controller
on Linux and to keep those fixes available for my own setup and anyone with similar issues.
Credit for the original bridge, architecture, and controller mappings belongs to the
original creator.

## Why this fork exists

The goal of this fork is to make the original bridge work better with the controller
behavior I encountered. The current changes include:

- **Additional device detection:** controller discovery and HID validation accept product
  IDs `0x3a05` and `0x3a08` under Corsair/Scuf vendor ID `0x1b1c`.
- **Stick magnitude limiting:** the input filter caps the combined X/Y stick magnitude at
  `30000`, scaling both axes together when input exceeds that limit.

These changes reflect my setup and the issues I experienced. Recognizing an additional
product ID does not guarantee that every controller variant uses the same input mappings.
If your device behaves differently, verify its events before assuming compatibility.

## What Scufpad does

Scufpad is a Linux application that bridges a physical Scuf Envision Pro controller to a
virtual Xbox Elite 2 controller through the kernel's `uinput` subsystem. The original
project handles the controller's non-standard evdev button and axis assignments so games
can receive standard Xbox-style input.

The bridge:

1. Reads events from the physical controller through evdev.
2. Translates its buttons and axes to Xbox mappings.
3. Applies stick and trigger deadzones and jitter filtering.
4. Emits events through a virtual Xbox Elite 2 controller
   (Microsoft VID `0x045e`, PID `0x0b12`).

It also grabs the controller's evdev interfaces to reduce duplicate input from the
physical device. Some games need additional SDL configuration, described below.

## Requirements

- Linux with the `uinput` kernel module and access to input devices.
- .NET 10 SDK to build or run from source.
- A compatible Scuf Envision Pro controller connected to the Linux host.
- For Native AOT publishing: the .NET native compilation prerequisites for your Linux
  distribution, including a C/C++ compiler toolchain and zlib development files.

Builds and the existing unit/integration suites can run without a connected controller.
Live bridging requires the physical controller and `/dev/uinput`; a cloud container
without device access cannot validate controller output.

## Quick start

Run these commands on the Linux machine connected to your controller.

### 1. Configure device permissions

Create udev rules for both product IDs recognized by this fork:

```bash
sudo tee /etc/udev/rules.d/99-scuf.rules > /dev/null << 'EOF'
SUBSYSTEM=="input", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="3a05", MODE="0666"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="3a05", MODE="0666"
SUBSYSTEM=="input", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="3a08", MODE="0666"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="3a08", MODE="0666"
KERNEL=="uinput", MODE="0666"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
sudo modprobe uinput
```

These rules grant all local users access to the matching devices. Use your distribution's
group-based permissions if you need more restricted access. Unplug and reconnect the
controller after applying the rules.

To load `uinput` automatically on boot:

```bash
echo 'uinput' | sudo tee /etc/modules-load.d/uinput.conf
```

### 2. Run the bridge

From the repository root:

```bash
dotnet run --project src/Scufpad/scufpad.csproj
```

When the controller is detected and the virtual device is created, the bridge prints
`Bridge service started. Press Ctrl+C to exit.` Keep it running while playing.

## Steam and SDL games

For Steam games, disable Steam Input for the game:

1. Right-click the game and open **Properties**.
2. Open **Controller**.
3. Set the override to **Disable Steam Input**.

For games using SDL, add this launch option so SDL ignores both physical device IDs
recognized by this fork:

```text
SDL_GAMECONTROLLER_IGNORE_DEVICES=0x1b1c/0x3a05,0x1b1c/0x3a08 %command%
```

For a non-Steam SDL game, set the same environment variable when launching the game.
Start the bridge first, then check that the game receives input from the virtual controller.

## Building and testing

Use the solution file explicitly from the repository root:

```bash
dotnet restore scufpad.slnx
dotnet build scufpad.slnx --no-restore
dotnet build scufpad.slnx -c Release --no-restore
dotnet test scufpad.slnx
```

Run individual test projects:

```bash
dotnet test tests/scufpad.Tests.Unit/scufpad.Tests.Unit.csproj
dotnet test tests/scufpad.Tests.Integration/scufpad.Tests.Integration.csproj
```

The integration suite exercises mapping/filter behavior and interop structures. It does
not replace testing a physical controller and virtual gamepad on a Linux host.

Publish a standalone Native AOT binary for Linux x64:

```bash
dotnet publish src/Scufpad/scufpad.csproj -c Release -r linux-x64 -o publish/
./publish/Scufpad
```

The published binary does not require an installed .NET runtime, but still needs controller
access and `uinput` on the host.

## Architecture

```text
Physical Scuf controller
        ↓
EvdevReader → InputPoller (4 ms polling timeout)
        ↓
EnvisionMapping
        ↓
InputFilter (deadzone, jitter filtering, stick magnitude limit)
        ↓
VirtualGamepad via uinput
        ↓
Games receive Xbox controller input
```

The implementation lives under `src/Scufpad/`, with directories for discovery, input,
interop, mapping, output, and bridge services. On V2 hardware, both triggers are read
through evdev; hidraw processing is not used by the main bridge loop.

## Hardware Mapping Reference

### Scuf Envision Pro V2 Axis Mappings

The V2 uses completely non-standard axis positions:

| Scuf Axis         | evdev Code     | Value Range     | Maps To       |
|-------------------|----------------|-----------------|---------------|
| Left Stick X      | ABS_X (0)      | -32768 to 32767 | Left Stick X  |
| Left Stick Y      | ABS_Y (1)      | -32768 to 32767 | Left Stick Y  |
| **Right Stick X** | **ABS_Z (2)**  | -32768 to 32767 | Right Stick X |
| **Left Trigger**  | **ABS_RX (3)** | 0 to 1023       | Left Trigger  |
| **Right Trigger** | **ABS_RY (4)** | 0 to 1023       | Right Trigger |
| **Right Stick Y** | **ABS_RZ (5)** | -32768 to 32767 | Right Stick Y |
| D-pad X           | ABS_HAT0X (16) | -1 to 1         | D-pad X       |
| D-pad Y           | ABS_HAT0Y (17) | -1 to 1         | D-pad Y       |

**Bold** = Non-standard mapping (differs from typical Xbox controllers)

### Scuf Envision Pro V2 Button Mappings

The V2 uses highly non-standard button codes:

| Scuf Button | evdev Code           | Standard Code | Maps To           |
|-------------|----------------------|---------------|-------------------|
| A           | BTN_SOUTH (0x130)    | BTN_SOUTH     | A Button          |
| B           | BTN_EAST (0x131)     | BTN_EAST      | B Button          |
| **X**       | **BTN_C (0x132)**    | BTN_WEST      | X Button          |
| Y           | BTN_NORTH (0x133)    | BTN_NORTH     | Y Button          |
| **LB**      | **BTN_WEST (0x134)** | BTN_TL        | Left Bumper       |
| **RB**      | **BTN_Z (0x135)**    | BTN_TR        | Right Bumper      |
| **L3**      | **BTN_TL2 (0x138)**  | BTN_THUMBL    | Left Stick Click  |
| **R3**      | **BTN_TR2 (0x139)**  | BTN_THUMBR    | Right Stick Click |
| Select      | BTN_TL (0x136)       | BTN_SELECT    | Back/Select       |
| Start       | BTN_TR (0x137)       | BTN_START     | Start             |
| Guide       | BTN_MODE (0x13c)     | BTN_MODE      | Guide/Home        |
| Paddle 1    | BTN_TRIGGER_HAPPY1   | -             | Paddle 1          |
| Paddle 2    | BTN_TRIGGER_HAPPY2   | -             | Paddle 2          |
| Paddle 3    | BTN_TRIGGER_HAPPY3   | -             | Paddle 3          |

**Bold** = Non-standard mapping

**Note:** The V2 only has 3 paddle buttons, not 4.

## Input filter defaults

| Setting | Value |
|---------|-------|
| Poll timeout | 4 ms |
| Stick deadzone | 3500 |
| Trigger deadzone | 10 |
| Stick jitter threshold | 300 |
| Trigger jitter threshold | 20 |
| Maximum combined stick magnitude in this fork | 30000 |

The deadzone and jitter thresholds are defined in `src/Scufpad/Mapping/InputFilter.cs`.

## Troubleshooting

### Controller is not detected

Check whether the host sees the controller and exposes its input devices:

```bash
lsusb
ls -la /dev/input/event*
```

Look for vendor ID `1b1c` and product ID `3a05` or `3a08`. Confirm the controller is
connected, then unplug/reconnect it after installing the udev rules.

### Permission denied or virtual gamepad creation fails

```bash
ls -la /dev/uinput /dev/input/event*
sudo modprobe uinput
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Check that `/etc/udev/rules.d/99-scuf.rules` includes your controller's product ID.
Containers also need the host to expose the devices; installing udev rules inside a
container alone will not provide that access.

### Double input or incorrect game behavior

Start the bridge before the game, disable Steam Input, and set the SDL ignore-device
launch option where applicable. Use `evtest` on the virtual Xbox controller to inspect
its output and distinguish bridge behavior from game-specific configuration.

When reporting an issue with this fork, include the controller's USB IDs, connection
mode, Linux distribution, game, and the observed input behavior.

## Credits and license

**Original creator:** [Chase Redmon](https://github.com/ChaseDRedmon), creator of
[the upstream Scufpad project](https://github.com/ChaseDRedmon/Scufpad). This fork retains
the original MIT license and copyright notice.

**Fork maintainer:** [Athae1218](https://github.com/Athae1218/Scufpad).
The changes here are intended to address issues I experienced with my own setup.

See [LICENSE](LICENSE) for the full MIT license.
