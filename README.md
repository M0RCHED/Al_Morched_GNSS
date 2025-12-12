# Al_Morched_GNSS
GNSS → BLE bridge using STM32 Bluepill. Interrupt-driven UART, clean STM32 HAL firmware, and real-time NMEA streaming from Quescan G10A-F32 to HC-08 BLE.


# 📡 STM32 Bluepill GNSS → BLE Bridge
### STM32F103C8T6 + Quescan G10A-F32 GNSS + HC-08 BLE

This repository contains a complete **GNSS-to-BLE bridge** built around an **STM32 Bluepill**.

The STM32 reads **NMEA sentences** from a **Quescan G10A-F32** GNSS module on USART2, validates them, filters useful messages (RMC / GGA), and forwards them to a smartphone using an **HC-08 BLE** module on USART1.

The goal of this project is to be:

- Reproducible: breadboard-friendly wiring and clear instructions  
- Professional: clean STM32 HAL code, interrupt-based design  
- Portfolio-ready: easy to showcase on GitHub and in interviews  

---

## 🔧 Project Overview

- MCU: STM32F103C8T6 (“Bluepill”)
- GNSS: Quescan G10A-F32 (NMEA output over UART)
- BLE: HC-08 (BLE 4.0, UART 3.3 V TTL)
- Firmware: C, STM32 HAL (STM32CubeIDE), bare-metal (no RTOS)
- Prototype: breadboard + jumper wires

Main idea:

> GNSS NMEA out → STM32 (parse, filter, validate) → BLE → Smartphone terminal/app

---

## 📚 Table of Contents

1. Features  
2. Hardware  
3. Wiring  
4. Configuring the HC-08 Baud Rate  
5. Firmware Architecture  
6. Building and Flashing  
7. Testing with Android  
8. Roadmap / Future Improvements  
9. License  

(Links will still work when GitHub renders this file.)

---

## ✅ Features

This firmware is designed to look and feel like a real embedded production module:

- Interrupt-driven UART reception on GNSS (USART2)
- Ping-pong (double) buffering for NMEA lines
- NMEA checksum validation before forwarding
- Filtering of NMEA sentence types, for example:
  - $GNRMC
  - $GNGGA
  - $GPRMC
  - $GPGGA
- Forwarding to BLE over USART1 at 115200 baud
- Status LED on PC13:
  - Toggles every N valid GNSS messages (simple activity indicator)

You can easily extend this to:

- Parse latitude / longitude / speed on the MCU
- Add commands from the phone back to the GNSS module
- Use an RTOS later (FreeRTOS, etc.)

---

## 🧩 Hardware

You will need:

- 1× STM32F103C8T6 Bluepill development board  
- 1× Quescan G10A-F32 GNSS module  
  - Example product link: https://www.aliexpress.com/item/1005007090607843.html
- 1× HC-08 BLE module (3.3 V TTL UART)
- 1× GNSS active antenna compatible with the module
- 1× USB-to-UART adapter (FT232 / CH340 / CP2102)
  - Used to configure the HC-08 baud rate via AT commands
- Breadboard + jumper wires
- USB cable for powering and flashing the STM32 (via ST-Link or similar)

Make sure all devices share a common ground (GND).

---

## 🔌 Wiring

The project uses:

- USART2 for GNSS
- USART1 for BLE
- PC13 as a status LED

### GNSS → STM32 (USART2)

- GNSS TX → STM32 PA3 (USART2 RX)
- Optional: GNSS RX → STM32 PA2 (USART2 TX) if you want to send commands
- GNSS VCC → 3.3 V or 5 V (check your GNSS breakout board)
- GNSS GND → STM32 GND

### HC-08 → STM32 (USART1)

- HC-08 TXD → STM32 PA10 (USART1 RX)
- HC-08 RXD → STM32 PA9  (USART1 TX)
- HC-08 VCC → 3.3 V or 5 V (check your module)
- HC-08 GND → STM32 GND
- HC-08 KEY or EN → depends on module; sometimes needed for AT mode

### LED

- On the Bluepill, the on-board LED is usually wired to PC13.
- This pin is configured as a digital output and toggled to show GNSS activity.

---

## 🔄 Configuring the HC-08 Baud Rate

Many HC-08 modules default to 9600 baud.  
In this project, the STM32 communicates with the HC-08 at 115200 baud, so we must reconfigure the module.

### Option A – Using a USB-to-UART Adapter (recommended)

1. Connect the module:

    Adapter TX → HC-08 RXD  
    Adapter RX → HC-08 TXD  
    Adapter GND → HC-08 GND  
    Adapter 5 V or 3.3 V → HC-08 VCC (according to your module)  
    HC-08 KEY or EN → set according to your module’s datasheet (some require it HIGH for AT mode)

2. Open a serial terminal (PuTTY, TeraTerm, etc.) with:

    Port: COM port of your adapter  
    Baud rate: 9600  
    Data bits: 8  
    Parity: None  
    Stop bits: 1  
    Flow control: None  

3. Enter AT mode:

    Type:

        AT

    You should see:

        OK

4. Change baud rate to 115200:

    The exact command depends on your HC-08 firmware version.  
    A common command is:

        AT+BAUD8

    On many HC-08 modules:

    - BAUD4 = 9600  
    - BAUD8 = 115200  

    After sending the command, you should see a confirmation like:

        OK+Set

5. Power cycle the module:

    Disconnect and reconnect power to the HC-08.

6. Verify at 115200 baud:

    - Re-open your serial terminal at 115200  
    - Send "AT" again  
    - Expected response: "OK"

Now your HC-08 is ready to work at 115200 baud with the STM32.

### Option B – Using the Bluepill as a bridge

If you do not have a USB-to-UART adapter, you can temporarily flash a simple firmware on the STM32 that forwards USB CDC ↔ UART1 and send AT commands through a PC terminal.  
Once the baud rate is configured, you can flash the GNSS bridge firmware.

---

## 🧠 Firmware Architecture

The firmware is written in C using the STM32 HAL libraries (STM32CubeIDE). It follows these steps:

1. Initialization  
   - Configure system clock  
   - Initialize GPIO (including PC13 LED)  
   - Initialize USART1 (BLE, 115200 baud)  
   - Initialize USART2 (GNSS, 115200 baud)  
   - Start UART receive interrupts for both USARTs  

2. Interrupt-based RX (USART2: GNSS)  
   - Bytes from GNSS arrive one by one  
   - An interrupt (HAL_UART_RxCpltCallback) is triggered for each byte  
   - The code appends the byte into a current line buffer  
   - When a newline character '\n' is received:  
     - The line is terminated with "\r\n"  
     - The buffer index is swapped (ping-pong buffers)  
     - A flag such as nmea_ready is set to notify the main loop  

3. Main loop  
   - When nmea_ready is set:  
     - Read the completed NMEA line from the buffer  
     - Validate its checksum  
     - Check if the sentence type is one of the desired types:  
       - $GNRMC  
       - $GNGGA  
       - $GPRMC  
       - $GPGGA  
     - If valid:  
       - Forward the line via USART1 to the HC-08  
       - Increment a counter and toggle the PC13 LED every N messages  

4. USART1 (BLE) RX  
   - The firmware can also receive bytes from the phone over BLE  
   - For now, the default behavior only re-arms the RX interrupt  
   - This can be extended to accept commands from the phone or update GNSS configuration  

This separation of concerns (ISR vs main loop) keeps the code:

- Responsive (interrupts handle incoming data)  
- Safe (heavy processing done outside interrupts)  
- Extensible (easy to add parsing, logging, and commands)

---

## 🛠 Building and Flashing

The project is intended for STM32CubeIDE.

1. Clone or download this repository:

        git clone https://github.com/your-username/bluepill-gnss-ble-bridge.git
        cd bluepill-gnss-ble-bridge

2. Open STM32CubeIDE:

   - File → "Open Projects from File System…"  
   - Select the "firmware" folder  
   - Finish the import

3. Check the configuration:

   - Target MCU is STM32F103C8T6 (or your Bluepill variant)  
   - USART1 and USART2 pins match your wiring (PA9/PA10, PA2/PA3)

4. Build the project:

   - Project → "Build Project"  

5. Flash the firmware:

   - Connect an ST-Link (or clone) to SWDIO / SWCLK / GND / 3.3 V  
   - Run → "Run As" → STM32 Cortex-M Application  
   or  
   - Run → "Debug As" → STM32 Cortex-M Application  

Once flashed, the Bluepill will:

- Listen to the GNSS on USART2  
- Forward valid NMEA messages to BLE on USART1  

---

## 📱 Testing with Android

To see the GNSS data on a phone:

1. Pair with HC-08:

   - Enable Bluetooth on your phone  
   - Search for devices  
   - Select "HC-08"  
   - Default PIN is often 1234 or 0000  

2. Install a serial terminal app:

   - Example: "Serial Bluetooth Terminal" (Android)

3. Connect to HC-08 from the app:

   - Choose HC-08 as the device  
   - Configure the connection at 115200 baud, 8 data bits, no parity, 1 stop bit  

4. Move to a location with GNSS coverage:

   - Near a window or outdoors  
   - Wait for the GNSS module to acquire a fix  

5. Observe NMEA output:

   You should see lines such as:

        $GNRMC, ...
        $GNGGA, ...

   The PC13 LED will toggle regularly as valid messages are processed.

If you see no data:

- Check power and wiring  
- Confirm baud rates (GNSS, HC-08, STM32)  
- Optionally connect a USB-UART adapter directly to the GNSS TX pin to verify that NMEA data is present  

---

## 🗺 Roadmap / Future Improvements

This project is intentionally kept simple but designed for extension. Some future ideas:

### Firmware

- Add a NMEA parser to extract:
  - Latitude and longitude  
  - Speed and heading  
  - Number of satellites and fix quality  
- Send a more compact format (JSON, binary, or UBX)  
- Add a command interface over BLE to:
  - Change GNSS update rate  
  - Enable or disable specific NMEA sentences  
- Port the application to FreeRTOS:
  - Task for GNSS receive and parse  
  - Task for BLE communication  
  - Task for status / logging  

### Hardware

- Move from breadboard to a custom PCB  
- Add ESD protection on UART and antenna lines  
- Add dedicated status LEDs (GNSS fix, BLE connected)  
- Add battery power and a charging circuit  

### Mobile / Cloud

- Simple Android app to:
  - Display position on a map  
  - Log tracks to GPX/KML files  
- Forward GNSS data to the cloud over MQTT or HTTP  
- Create a small web dashboard to visualize tracks  

---

## 📄 License

This project is released under the MIT License.  
You are free to use, modify, and distribute it, including for commercial purposes, as long as the license notice is preserved.
