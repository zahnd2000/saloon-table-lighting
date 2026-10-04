# saloon-table-lighting

ESP controlled battery driven LED strips for our saloon tables

This program will run on a ESP32-S3 microcontroller using **ESP-IDF (C++)**, not Arduino framework.

---

## ESP-IDF + C++ Notes

- **Build**: CMake-based (`idf.py build`), no Arduino IDE
- **Drivers**: Use ESP-IDF native C drivers (`driver/rmt_tx.h`, `driver/spi_master.h`, `driver/uart.h`)
- **C++**: Wrap C handles in RAII classes; ESP-IDF supports C++ (`.cpp` files, `extern "C"` for C headers)
- **Components**: Place reusable code in `components/` with own `CMakeLists.txt` + `idf_component_register()`
- **Memory**: 512 KB SRAM — use `MALLOC_CAP_DMA` for SPI/RMT DMA buffers
- **No Arduino libs**: `FastLED`, `Adafruit_NeoPixel` unavailable — write custom RMT encoder / SPI transmitter
- **Example reference**: ESP-IDF `examples/peripherals/rmt/led_strip` and `examples/peripherals/spi_master`

---

## LED Protocols for ESP32-S3

### 1. WS2812 / NeoPixel / SK6812 (Single-Wire)
- **Protocol**: Single data line, self-clocking at 800 kHz (WS2812B) or 400 kHz (WS2812S)
- **ESP32-S3 Support**: Native via RMT (Remote Control Transceiver) peripheral — zero CPU overhead, precise timing
- **Channels**: RGB or RGBW (SK6812)
- **Max LEDs**: ~1000+ per channel (limited by RAM for frame buffer)
- **Wiring**: 3 wires (VCC, GND, Data). **Level shifter recommended** (3.3V → 5V) for reliable operation at 5V strips
- **Power**: 5V typical, ~60 mA/LED max (white)
- **ESP-IDF Driver**: `driver/rmt_tx.h` — `rmt_new_tx_channel()`, `rmt_transmit()`, `rmt_encoder_t` for WS2812 encoding
- **Components**: `esp-idf-lib` `rmt_ws2812`, or write custom RMT encoder (see `led_strip` component in ESP-IDF examples)
- **C++**: Wrap C driver in RAII class; use `std::vector<uint8_t>` for frame buffer

### 2. APA102 / SK9822 / DotStar (SPI)
- **Protocol**: Standard SPI (Clock + Data), up to 20+ MHz
- **ESP32-S3 Support**: Native hardware SPI — extremely reliable, no timing constraints
- **Channels**: RGB
- **Max LEDs**: Very high (limited by SPI speed and refresh rate)
- **Wiring**: 4 wires (VCC, GND, CI, DI). **No level shifter needed** — 3.3V logic works directly
- **Power**: 5V typical, ~60 mA/LED max
- **Advantages**: No timing issues, global brightness register, faster updates
- **ESP-IDF Driver**: `driver/spi_master.h` — `spi_bus_initialize()`, `spi_bus_add_device()`, `spi_device_transmit()`
- **DMA**: Use `SPICOMMON_BUSFLAG_MASTER | SPICOMMON_BUSFLAG_GPIO_PINS` with DMA channel for zero-copy
- **C++**: RAII wrapper for `spi_device_handle_t`; frame buffer as `std::vector<uint8_t>` (4 bytes/LED: start frame + RGB + end frame)

### 3. WS2815 / GS8208 (Dual-Signal, 12V)
- **Protocol**: Same timing as WS2812 but with backup data line (DI + BI)
- **Voltage**: 12V — can run longer strips with less voltage drop
- **ESP32-S3 Support**: Same as WS2812 via RMT
- **Wiring**: 3 wires (12V, GND, Data). Level shifter needed for data line
- **Use case**: Longer runs (5m+) without power injection

### 4. TM1814 / TM1829 / UCS2904 (SPI-like)
- **Protocol**: Clock + Data, similar to APA102 but different frame format
- **ESP32-S3 Support**: Hardware SPI
- **Voltage**: 12V or 24V variants available
- **Less common**, check library support

### 5. DMX512 (RS-485) — For Professional Fixtures
- **Protocol**: Differential RS-485, 250 kbps, up to 512 channels/universe
- **ESP32-S3 Support**: UART + external RS-485 transceiver (e.g., MAX485, SN75176)
- **Hardware needed**: RS-485 transceiver IC + isolation recommended
- **Use case**: Controlling commercial LED fixtures, moving lights, dimmers
- **ESP-IDF Driver**: `driver/uart.h` — `uart_driver_install()`, `uart_write_bytes()` with 250000 baud, 8N2
- **Timing**: Break (88µs) + MAB (8µs) + 513 slots (4µs each) — use UART pattern detection or timer for break/MAB
- **Components**: `esp-dmx` component (C), wrap in C++ class

### 6. Art-Net / sACN (Ethernet / WiFi)
- **Protocol**: UDP-based lighting protocols over IP (Art-Net: 6454, sACN: 5568)
- **ESP32-S3 Support**: Native WiFi + Ethernet (via RMII/GMAC with external PHY like LAN8720)
- **Hardware needed**: Ethernet PHY for wired; WiFi works but less reliable for high channel counts
- **Use case**: Large installations, integration with lighting consoles (GrandMA, ChamSys, etc.)
- **ESP-IDF**: `esp_netif`, `lwip` raw UDP sockets (`lwip/sockets.h`) — `socket()`, `bind()`, `recvfrom()`
- **WiFi**: `esp_wifi` + `esp_event` for STA/AP mode
- **Ethernet**: `esp_eth` + `esp_eth_phy_lan8720` for wired
- **Parsing**: Custom C++ parsers for Art-Net (OpCode 0x5000) / sACN (E1.31) — no heavy libraries needed

---

## ESP32-S3 Specific Advantages

| Feature | Benefit for LED Control |
|---------|------------------------|
| **RMT peripheral** (8 channels) | Drive up to 8 independent WS2812 strips simultaneously with zero CPU load |
| **Hardware SPI** (3 controllers) | Drive multiple APA102 strips at high speed |
| **DMA** | Large frame buffers without CPU copy overhead |
| **160/240 MHz CPU** | Plenty of headroom for effects, WiFi, Bluetooth |
| **512 KB SRAM** | Frame buffers for 3000+ RGB LEDs |
| **USB OTG** | Direct firmware updates, serial debug |
| **Touch/I2C/I2S** | Sensors, audio-reactive, display co-processors |

---

## Recommended Hardware Additions

| Component | Purpose | Notes |
|-----------|---------|-------|
| **Level shifter** (74AHCT125, TXS0108E) | 3.3V → 5V logic for WS2812 | Required for reliable 5V strip control |
| **Logic-level MOSFET** (IRLZ44N) | High-side power switching | For battery-powered projects to cut strip power |
| **Fuse / PTC** | Overcurrent protection | 3–5A per power injection point |
| **Capacitor** (1000 µF, 6.3V/10V) | Power supply smoothing | Across VCC/GND at strip start |
| **Resistor** (330–470 Ω) | Data line protection | In series on data line near ESP |
| **Buck converter** | 5V/12V from battery | If running from Li-ion (3.7V) or 12V lead-acid |
| **ESD protection** (TVS diodes) | Data line protection | For hot-pluggable strips |

---

## Quick Decision Guide

| Priority | Choose |
|----------|--------|
| Simplicity, low cost, 5V strips | **WS2812/SK6812** + RMT + level shifter |
| Reliability, no timing hassles, 3.3V logic OK | **APA102/SK9822** + hardware SPI |
| Long runs (5m+), fewer power injection points | **WS2815** (12V) + RMT + level shifter |
| Professional fixtures, DMX gear | **DMX512** + MAX485 + UART |
| Large scale, networked, console integration | **Art-Net/sACN** + Ethernet PHY |

---

## Example: Minimal WS2812 Setup (ESP32-S3)

```
ESP32-S3          Level Shifter         WS2812 Strip
──────────────────────────────────────────────────────
GPIO 8 (RMT) ───► A1 (3.3V side)        DI
GND ────────────► GND ─────────────────► GND
3.3V ───────────► VCCA (3.3V)           ─── (optional)
5V (USB/VBUS) ──► VCCB (5V) ────────────► VCC
                │
                └─► OE tied to GND (always enabled)
```

Add 470 Ω resistor on data line, 1000 µF cap across strip VCC/GND.

---

## Next Steps

1. Choose strip type (WS2812 vs APA102 vs WS2815)
2. Calculate power budget (LEDs × 60 mA × 5V/12V)
3. Design power distribution (injection points every 1–2m)
4. Select battery + buck/boost converter
5. Prototype on breadboard with level shifter
6. Implement ESP-IDF components:
   - WS2812: Custom RMT encoder (`rmt_encoder_t`) + `rmt_transmit()`
   - APA102: SPI master + DMA buffer
   - Effects: C++ classes with frame buffer (`std::vector<uint8_t>`), double-buffered for flicker-free
7. Build system: CMake (`idf_component_register`, `CMakeLists.txt`)