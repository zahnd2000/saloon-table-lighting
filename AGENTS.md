# saloon-table-lighting — Agent Instructions

## Project Overview
ESP-IDF C++ project for ESP32-S3 controlling battery-driven LED strips on saloon tables. **Not Arduino** — uses native ESP-IDF C drivers with C++ RAII wrappers.

## Build & Flash
```bash
idf.py build          # Build firmware
idf.py flash          # Flash to device
idf.py monitor        # Serial monitor (Ctrl+] to exit)
idf.py build flash monitor  # All in one
```

## Key Constraints
- **No Arduino libs**: `FastLED`, `Adafruit_NeoPixel` unavailable
- **Memory**: 512 KB SRAM — use `MALLOC_CAP_DMA` for SPI/RMT DMA buffers
- **Drivers**: Native ESP-IDF C headers (`driver/rmt_tx.h`, `driver/spi_master.h`, `driver/uart.h`)
- **C++**: Wrap C handles in RAII classes; `extern "C"` for C headers

## LED Strip Protocols (choose one)
| Type | Interface | ESP32-S3 Driver | Notes |
|------|-----------|-----------------|-------|
| WS2812/SK6812 | RMT (8 ch) | `driver/rmt_tx.h` | Needs 3.3V→5V level shifter |
| APA102/SK9822 | SPI (3 ctrl) | `driver/spi_master.h` | 3.3V logic OK, no shifter |
| WS2815/GS8208 | RMT | Same as WS2812 | 12V, longer runs |
| DMX512 | UART + RS-485 | `driver/uart.h` | 250 kbps, 8N2, break/MAB timing |
| Art-Net/sACN | WiFi/Ethernet | `esp_netif`, `lwip` | UDP 6454/5568 |

## Component Structure
Reusable code goes in `components/` with own `CMakeLists.txt` + `idf_component_register()`.

## Reference Examples
- ESP-IDF: `examples/peripherals/rmt/led_strip`
- ESP-IDF: `examples/peripherals/spi_master`

## Hardware Requirements
- Level shifter (74AHCT125) for WS2812 data line
- 470 Ω resistor on data line near ESP
- 1000 µF capacitor across strip VCC/GND
- Logic-level MOSFET for power switching (battery projects)
- Fuse/PTC (3–5A per injection point)

## Development Notes
- Target: ESP32-S3 (8 RMT channels, 3 SPI controllers, DMA, 160/240 MHz, 512 KB SRAM, USB OTG)
- Frame buffers: `std::vector<uint8_t>`, double-buffered for flicker-free effects
- WS2812: Custom `rmt_encoder_t` + `rmt_transmit()`
- APA102: SPI master + DMA, 4 bytes/LED (start frame + RGB + end frame)