<img src="https://github.com/bdring/FluidNC/wiki/images/logos/FluidNC.svg" width="600">

---

## 📢 CostyCNC Custom Mod: Universal SD AutoRun Port

**This repository is a customized fork of FluidNC that introduces standalone plug-and-play SD Card AutoRun automation.** Inserting a MicroSD card immediately triggers and streams the G-code job without a computer or screen connected.

### 🛠 Refactoring & Object-Oriented Migration
* **Framework Porting & Thread Isolation (`poll_once` Hook):** FluidNC coordinates multitasking via strict FreeRTOS core isolation. Hardware logs and streams loop asynchronously inside `poll_once()` on Core 0, while G-code streaming consumes `cmd_queue` bounds on Core 1. To inject our custom engine without mutating task priorities or creating memory collisions, the interactive macro routine was hooked directly into the non-blocking idle checks of `poll_once()` inside **`FluidNC/src/Protocol.cpp`** starting at **line 329**, upstream of standard input polling loops.

             if (!Job::active()) {
            // No job: every line goes to cmd_queue.  Gate on queue room so a
            // slow consumer bounds read-ahead - the flow control the old
            // single slot gave.
            if (uxQueueSpacesAvailable(cmd_queue)) {
                
                // --- INIZIO IMPLEMENTAZIONE COSTYCNC AUTORUN LOGIC ---
                static bool file_already_started = false;
                static bool system_just_powered_on = true; // BOOT SAFETY GUARD
                static uint32_t last_sd_check_time = 0;

                // ASYNCHRONOUS 3-SECOND SCHEDULER
                if (millis() - last_sd_check_time >= 3000) {
                    last_sd_check_time = millis();

                    // Verifica se l'oggetto SDCard è configurato e se la scheda è montata
                    bool current_sd_ready = (config->_sdCard != nullptr && config->_sdCard->is_mounted()); 

                    if (!current_sd_ready) {
                        // Se la scheda non c'è o viene rimossa, azzeriamo i flag di blocco
                        file_already_started = false;
                        system_just_powered_on = false; // La rimozione sblocca la sicurezza di boot
                    } 
                    else if (state_is(State::Idle) && !file_already_started && !system_just_powered_on) {
                        // In FluidNC il filesystem SD è accessibile tramite l'oggetto globale di archiviazione
                        fs::File root = config->_sdCard->open("/", "r");
                        if (root) {
                            fs::File file = root.openNextFile();
                            while (file) {
                                if (!file.isDirectory()) {
                                    const char* name = file.name();
                                    size_t len = strlen(name);
                                    
                                    // DYNAMIC FILE DETECTION: Cerca il prefisso '-' ed estensione '.nc'
                                    if (name[0] == '-' && len > 3 && strcmp(name + len - 3, ".nc") == 0) {
                                        log_info("CostyCNC AutoRun: Found target file " << name);
                                        
                                        // Genera ed inietta in modo sicuro il comando nell'inputBuffer/cmd_queue
                                        char cmd_buf[Channel::maxLine];
                                        snprintf(cmd_buf, sizeof(cmd_buf), "$SD/Run=/%s", name);
                                        
                                        // Invia il comando simulando l'input dalla Console principale
                                        cmd_queue_defer(cmd_buf, Console); 
                                        
                                        file_already_started = true;
                                        file.close();
                                        break;
                                    }
                                }
                                file.close();
                                file = root.openNextFile();
                            }
                            root.close();
                        }
                    }
                }
                // --- FINE IMPLEMENTAZIONE COSTYCNC AUTORUN LOGIC ---

                poll_input_channel();
            }
        } else {

* **Thread-Safe Architecture (`cmd_queue_defer` Injection):** Instead of forcing hardware-level macros or using unbuffered string injections, the automation safely interacts with FluidNC's dual-core task synchronization. It queries the dedicated SD object class (`config->_sdCard->is_mounted()`) on the storage stack and uses the native `cmd_queue_defer(cmd_buf, Console)` method to queue G-code execution. This pipes the automated commands directly into the thread-safe core processing ring, preventing memory collisions between the poller task (Core 0) and the motion controller (Core 1).

🐛 Bugs Found & Resolved

☁️ DevOps & Pipeline Fixes
* **BUG #1: 403 Forbidden Release Error** ➔ *Resolved*. The original pipeline crashed at the deployment step because the automatic token lacked explicit write access. Added `contents: write` permissions under the build job.
* **BUG #2: Broken Cache Resource Drain** ➔ *Resolved*. The workflow original key targeted `lockfiles` which FluidNC does not use, forcing an unsustainable 13m 17s download loop on every single commit. Realigned the cache key to target `platformio.ini`, slashing build times down to 4m 21s.
* **BUG #3: Downstream Deploy Credential Crash** ➔ *Resolved*. The automated script ended with a fatal `git push` crash (`fatal: could not read Username for 'https://github.com'`) because it tried to forward binaries to an external repository via an unavailable `RELEASE_COPY_TOKEN`. Commented out the external distribution block to keep release deployments inside your personal fork.
* **BUG #4: PlatformIO Environment NameError (`Import("env")` Crash)** ➔ *Resolved*. Running `build_merged.py` directly via a standard Python command outside of PlatformIO caused a pipeline crash because the global `Import()` function was undefined. Resolved by bypassing the script and utilizing native `esptool.py merge_bin` instructions to fuse the binaries directly via CLI.

⚙️ Pipeline & Core Firmware Modifications
* **MOD #5: Native Compiler Pruning (`manual.yml` PlatformIO Bypass)** ➔ *Added*. FluidNC's native `build-release.py` script relies on a hardcoded matrix that forces consecutive builds for multiple unused processor architectures (S3, Bluetooth, and non-radio versions). To maximize efficiency and reclaim free-tier cloud minutes, we bypassed the deployment script in `manual.yml` entirely, substituting it with a direct, single-target execution command (`platformio run -e wifi`). This restricts the build cycle strictly to the standard Wi-Fi stack used by the MKS-DLC32 motherboard, cutting downstream processing times to seconds.
* **MOD #6: Single-File Monolithic BIN Generation (`esptool.py merge_bin`)** ➔ *Added*. To ensure production-ready flashing on empty MKS-DLC32 boards and avoid script dependency crashes, the workflow leverages Python's native `esptool` stack to merge the bootloader, partition tables, bootapp data, and the custom Wi-Fi firmware into a single monolithic `.bin` file mapped from address **`0x0`**.
* **BUG #7: Dangerous Power-On Execution Loop** ➔ *Resolved*. If the CNC was turned on with an SD card already inside, the core logic would immediately trigger the motor drivers. Implemented a boot safety guard that forces an explicit, physics-based card insertion event before activating the engine.
* **BUG #8: Hardcoded Filename Restriction** ➔ *Resolved*. The original implementation locked users into a static `costycnc.nc` filename. Rewrote the file parser to dynamically search the FAT32 root directory for the first file beginning with a `-` prefix and ends with `.nc` (e.g., `-sedia.nc`).


---

### ⚙️ Automation Core Rules & Asynchronous Scheduling
* **Asynchronous 3-Second Scheduler:** The slot status is polled using an asynchronous internal clock interval (`millis() - last_sd_check_time >= 3000`) to protect CPU cycles. Because this loop runs independently in the background, execution is asynchronous: if you insert the card right before the 3-second window closes, **the cut starts instantly**. If inserted right after, it will wait for the next background interval to trigger.
* **The G-Code Modal Loop Warning (S0 / M03):** If your G-code drops power via `S0` or leaves `M03` active at the end without sending an explicit `M05`, the firmware state machine will get stuck in modal limbo. The asynchronous scheduler will notice an unstable state mismatch and re-execute the job endlessly. **Mandatory Rule:** Every G-code file **MUST conclude with an `M05` command** to return the machine to a stable `Idle` state.

---

### 🚀 Quick Start Guide

1. **Prepare the File:** Generate your G-code toolpath using your preferred CAM software (LightBurn, LaserGRBL, Aspire, etc.). Rename the target file by adding a single `-` character at the very beginning of the filename (Example: `sedia.nc` ➔ `-sedia.nc`).
2. **Setup the Media:** Copy the renamed file directly into the root directory of a **FAT32** formatted MicroSD card.
3. **Power On Safely:** Turn on your CNC machine. If the MicroSD card is already inside the slot during power-on or reboot, the boot safety guard activates and **nothing will happen**. The machine sits quietly in a stable `Idle` state.
4. **Trigger via Card Insertion:** Physically push the MicroSD card into the slot. The firmware polls the hardware slot asynchronously every 3 seconds:
   * **Instant Start:** If you insert the card right as the background 3-second CPU interval is expiring, the firmware reads it immediately and the cutting engine starts **instantly**.
   * **Delayed Start (Up to 3 Seconds):** If you insert the card right after a new loop interval has just started, you will experience a brief, synchronous internal clock delay (up to 3 seconds max) before the file is validated and streamed.
5. **Changing Jobs:** Once execution finishes, clear the system by removing the card. To run a new job, move the `-` prefix to a different file on your computer and re-insert the card into the machine.

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
