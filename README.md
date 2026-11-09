<img src="https://github.com/bdring/FluidNC/wiki/images/logos/FluidNC.svg" width="600">

---

## 📢 CostyCNC Custom Mod: Universal SD AutoRun Port

**This repository is a customized fork of FluidNC that introduces standalone plug-and-play SD Card AutoRun automation.** Inserting a MicroSD card immediately triggers and streams the G-code job without a computer or screen connected.

### 🛠 Refactoring & Cloud Optimization
* **The Pipeline Fix:** Fixed the GitHub Actions `manual.yml` workflow by adding explicit `contents: write` permissions to bypass the 403 authorization error. Realigned the broken cache structure from `lockfiles` to `platformio.ini`, dropping test build times from **13m 17s down to 4m 21s**.
* **Object-Oriented Migration:** Ported the standalone routine from legacy procedural Grbl_ESP32 into FluidNC's modern framework (**`FluidNC/src/Main.cpp`**), routing commands safely into the thread-safe `inputBuffer` using native `sd_card_is_mounted()` and `ff_exists()` hooks.

### ⚙️ Actual Core Logic & Safety Rules
* **Boot Safety Guard:** If the machine is powered on or rebooted with an SD card already inside the slot, **nothing happens**. The CNC stays safely in `Idle` to prevent accidental spindle/laser starts. The cycle triggers **only** on a physical card insertion.
* **Dynamic File Detection:** Instead of a fixed filename, the firmware scans the root directory for the **first available file that begins with a `-` prefix and ends with `.nc`** (e.g., `-sedia.nc`).
* **Asynchronous 3-Second Scheduler:** The slot status is polled using an asynchronous internal clock interval (`millis() - last_sd_check_time >= 3000`) to protect CPU cycles. Because this loop runs independently in the background, execution is asynchronous: if you insert the card right before the 3-second window closes, **the cut starts instantly**. If inserted right after, it will wait for the next background interval to trigger.

### 🐛 The G-Code Modal Loop Bug (S0 / M03 Warning)
If your G-code file drops power via `S0` or leaves `M03` active at the end without sending an explicit `M05`, the firmware state machine will get stuck in a modal limbo. The asynchronous 3-second scheduler will see the card, notice an unstable state mismatch, and re-execute the job endlessly.
* **Mandatory Rule:** Every automated G-code file used with this fork **MUST conclude with an `M05` command** to return the machine to a stable, static `Idle` state.

### 🚀 Quick Start Guide
1. Rename the file you want to execute by adding a single `-` at the beginning (e.g., `sedia.nc` ➔ `-sedia.nc`).
2. Copy it to the root directory of a **FAT32** MicroSD card.
3. Power up the machine (it will sit in *Idle*). 
4. **Insert the MicroSD card:** Execution depends on the internal clock phase; the firmware will locate your `-*.nc` file and start the cycle either **instantly or within a 3-second window**.
5. To change jobs, remove the card, swap the `-` prefix to a different file on your PC, and re-insert.

---


## Introduction

**FluidNC** is a CNC firmware optimized for the ESP32 controller. It is the next generation of firmware from the creators of Grbl_ESP32. It includes a web based UI and the flexibility to operate a wide variety of machine types. This includes the ability to control machines with multiple tool types such as laser plus spindle or a tool changer.  

## Firmware Architecture

- Object-Oriented hierarchical design
- Hardware abstraction for machine features like spindles, motors, and stepper drivers
- Extensible - Adding new features is much easier for the firmware as well as gcode senders.

## Machine Definition Method

There is no need to compile the firmware. You use an installation script to upload the latest release of the firmware and then create [config file](http://wiki.fluidnc.com/en/config/overview) text file that describes your machine.  That file is uploaded to the FLASH on the ESP32 using the USB/Serial port or WIFI.

You can have multiple config files stored on the ESP32. The default is config.yaml, but you can change that with [**$Config/Filename=<myOtherConfig.yaml>**](http://wiki.fluidnc.com/en/features/commands_and_settings#config_filename)

### AI Config File Helpers

Writing a config file by hand (or with the help of an AI assistant) can be error-prone given how many sections and fields FluidNC supports. The [`tools/`](tools/) directory has a formal spec, a JSON Schema, a command-line validator, and an MCP server that an AI assistant can call directly to check its own work. See [AI Config File Helpers](http://wiki.fluidnc.com/en/config/overview#ai-config-file-helpers) on the wiki for details.

## Basic Grbl Compatibility

The intent is to maintain as much Grbl compatibility as possible. It is 100% compatible with the day to day operations of running gcode with a sender, so there is no change to the Grbl gcode send/response protocol, and all Grbl gcode are supported. Most of the $ settings have been replaced with easily readable items in the config file.


## WebUI

FluidNC includes a built-in browser-based Web UI (Esp32_WebUI) so you control the machine from a PC, phone, or tablet on the same Wifi network.

## Wiki

[Check out the wiki](http://wiki.fluidnc.com) if you want the learn more about the feature or how to use it.

## Credits

The original [Grbl](https://github.com/gnea/grbl) is an awesome project by Sungeon (Sonny) Jeon. I have known him for many years and he is always very helpful. I have used Grbl on many projects.

The Wifi and WebUI is based on [this project.](https://github.com/luc-github/ESP3D-WEBUI)  

## Discussion

<img src="http://wiki.fluidnc.com/discord-logo_trans.png" width="180">

We have a Discord server for the development this project. Ask for an invite


## Donations

This project requires a lot of work and often expensive items for testing. Please consider a safe, secure and highly appreciated donation via the PayPal link below or via the GitHub sponsor link at the top of the page.

[![](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/donate/?hosted_button_id=8DYLB6ZYYDG7Y)
