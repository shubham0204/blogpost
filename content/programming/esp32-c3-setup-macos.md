---
title: Setting up the programming environment for ESP32-C3 on macOS
tags:
  - programming
  - embedded-systems
---
## Installing the ESP-IDF Installation Manager (`eim`)
To install ESP-IDF on macOS, we first need [ESP-IDF Installation Manager (`eim`)](https://docs.espressif.com/projects/idf-im-ui/en/latest/)that helps us install and manage different versions of ESP-IDF on our system. For macOS, we follow the recommended method of using Homebrew for installation:

```
brew install libgcrypt glib pixman sdl2 libslirp dfu-util cmake python
```

```
brew tap espressif/eim
brew install --cask eim-gui
```

An application named `eim` will appear in the `~/Applications` accessible from Spotlight and the Applications menu. Open it and select `Easy Installation`:

![[Pasted image 20260404071045.png]]

The latest stable version of ESP-IDF should be displayed along with the installation path:
![[Pasted image 20260404071113.png]]
Upon completion, the installed version should be visible in `eim`:
![[Pasted image 20260419072645.png]]

## Setting up VSCode for ESP32 Development
Install the [ESP-IDF extension in VSCode](https://marketplace.visualstudio.com/items?itemName=espressif.esp-idf-extension):
![[Pasted image 20260404070621.png]]

Open the command palette (`Cmd + Shift + P`) and select the `ESP-IDF: New Project` action:
![[Pasted image 20260404072428.png]]

Select the default ESP-IDF installation that we made in the previous steps:
![[Pasted image 20260419073607.png]]

A project wizard appears with pre-built templates and examples:
![[Pasted image 20260419073745.png]]

We choose the `hello_world` example present in `get-started`. In the `Create Project` pane, select the `ESP-IDF Target` as `esp32c3`:
![[Pasted image 20260419074120.png]]
The following files are created where `hello_world_main.c` holds the source code:
![[Pasted image 20260419074242.png]]

## Running code on the ESP32-C3
Connect your ESP32-C3 via a USB C cable to the Mac. A glowing red LED indicates that is receiving power:
![[Pasted image 20260419080358.png]]

The device target in the bottom left corner of VSCode should show the `esp32c3`:
![[Pasted image 20260419080557.png]]
If not shown, click device target label and select the device manually:
![[Pasted image 20260419080637.png]]

Select the `ESP-IDF: Build, Flash and Start a Monitor` action from the command palette
![[Pasted image 20260419080751.png]]

This action will start building the source code for `esp32c3`:
![[Pasted image 20260419080955.png]]
Display information about the executable that will be flashed on the device:
![[Pasted image 20260419081028.png]]
And asks for a flash method. We select `JTAG` and proceed:
![[Pasted image 20260419081054.png]]

Allow the execution of the `OpenOCD server`:
![[Pasted image 20260419081126.png]]

The monitor starts and the `hello world` message is visible on the console:
![[Pasted image 20260419081321.png]]





