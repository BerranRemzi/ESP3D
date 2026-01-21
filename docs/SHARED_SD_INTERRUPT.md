# Shared SD Card with Interrupt-based SS Detection

## Overview

This feature enables ESP8266/ESP32 to share an SD card with a printer using interrupt-based Slave Select (SS) detection. The implementation uses hardware interrupts to monitor when the printer is accessing the SD card, ensuring conflict-free access.

## Hardware Configuration

### Pin Assignment (ESP8266)
- **Pin 4 (D2)**: Input for SS detection from printer (with pull-up)
- **Pin 5 (D1)**: SS output for ESP8266 control
- **1kΩ resistors**: Connect between printer and SD card lines for sharing

### How It Works

1. The ESP8266 monitors Pin 4 using a falling edge interrupt
2. When the printer's SS line goes LOW (active), the interrupt handler sets a flag
3. Before accessing the SD card, ESP8266 checks:
   - The interrupt flag (`_printer_accessing_sd`)
   - A direct read of Pin 4 as a backup check
4. If the printer is using the SD card, ESP8266 waits until it's available

## Configuration

### Enable Shared SD in `configuration.h`:

```cpp
// Enable shared SD card connection
#define SD_DEVICE_CONNECTION ESP_SHARED_SD

// Choose SD library (ESP8266 supports both)
#define SD_DEVICE ESP_SD_NATIVE  // or ESP_SDFAT2

// Pin definitions (optional - defaults to pins 4 and 5)
// #define ESP_SD_CS_SENSE 4   // Input from printer SS
// #define ESP_SD_CS_PIN 5     // Output SS for ESP8266
```

### For Custom Pin Assignment:

In `myconfig.h` or `configuration.h`:

```cpp
#define ESP_SD_CS_SENSE 4    // Pin for SS input detection
#define ESP_SD_CS_PIN 5      // Pin for SS output control
```

## Features

### Interrupt-Based Detection
- **Fast Response**: Hardware interrupt detects printer access immediately
- **Low Overhead**: No polling required, minimal CPU usage
- **Reliable**: Uses CHANGE interrupt to catch both rising and falling edges
- **20-Second Timeout**: SD remains blocked for 20 seconds after printer access
- **User Feedback**: Shows dummy file when SD is blocked by printer

### Conflict Prevention
- Checks interrupt flag before enabling SD access
- Enforces 20-second timeout after printer activity
- Shows "SD card is used by printer.txt" when blocked
- Properly manages SPI bus sharing between ESP and printer

### Automatic Management
- Interrupt automatically attached during SD initialization
- Interrupt automatically detached during SD cleanup
- Thread-safe volatile flag for interrupt handler

## Implementation Details

### Key Functions

#### `sdCsInterrupt()` (ISR)
Interrupt Service Routine that:
- Runs in IRAM for fast execution
- Updates `_printer_accessing_sd` flag
- Tracks timestamp (`_last_printer_access_time`) when printer accesses SD
- Checks if CS is LOW (active) or HIGH (inactive)

#### `isSDBlockedByPrinter()`
Checks if SD is blocked by printer:
- Returns `true` if printer is actively using SD
- Returns `true` if within 20-second timeout after last printer access
- Returns `false` only after timeout expires

#### `getBlockedTimeRemaining()`
Returns remaining time (in milliseconds) that SD is blocked:
- Returns full 20000ms if printer is actively using SD
- Returns remaining time if within timeout period
- Returns 0 if SD is available

#### `enableSharedSD()`
Before taking SD access:
- Checks if SD is blocked (via `isSDBlockedByPrinter()`)
- Logs remaining timeout time for debugging
- Only enables SD if timeout has expired

#### `ESP_SD::open()`
When opening root directory ("/"):
- If SD is blocked by printer, returns dummy file
- Dummy file named "SD card is used by printer.txt"
- Shows in file listings when SD is unavailable
- Prevents access errors and provides user feedback

## Example Usage

The feature is transparent to users when properly configured:

```cpp
// In your ESP3D code, SD access works normally:
if (ESP_SD::accessFS()) {
    ESP_SDFile file = ESP_SD::open("/test.gcode", ESP_FILE_READ);
    // ... use file ...
    file.close();
    ESP_SD::releaseFS();
}
```

The interrupt-based detection ensures:
1. `accessFS()` returns `false` if printer is using SD
2. SD is only accessed when safe
3. No corruption or conflicts occur

## Debugging

Enable logging to see interrupt activity:

```
Attached interrupt to SD CS sense pin 4
Enable Shared SD if possible
Printer is accessing SD (detected via interrupt), skip
```

## Compatibility

- **ESP8266**: Fully supported with both ESP_SD_NATIVE and ESP_SDFAT2
  - All GPIO pins except GPIO16 support interrupts
  - Pin 4 (D2) is confirmed to support interrupts
- **ESP32**: Fully supported with all SD implementations
  - All GPIO pins support interrupts
  - Pin 4 is confirmed to support interrupts
- **Backwards Compatible**: Works with existing FYSETC WiFi Pro configuration

## Pin Selection Notes

### Recommended Pins
- **ESP8266**: Use GPIO 4 (D2) for CS sense, GPIO 5 (D1) for CS control
- **ESP32**: Any GPIO pin can be used, but maintain consistency

### Pins to Avoid
- **ESP8266 GPIO16**: Does not support interrupts, do not use for CS sense
- **ESP32 Strapping Pins**: GPIO 0, 2, 5, 12, 15 - avoid if possible for stability

## Notes

1. The interrupt uses CHANGE mode to detect both rising and falling edges
2. The ISR is marked with IRAM_ATTR for ESP8266/ESP32 compatibility
3. The `_printer_accessing_sd` flag is volatile for thread safety
4. Pull-up resistor on sense pin prevents false triggers

## Testing

To verify the implementation:

1. Connect hardware with 1kΩ resistors
2. Enable shared SD in configuration
3. Monitor serial output for interrupt messages
4. Try accessing SD while printer is active
5. Verify ESP8266 waits for printer to finish

## Troubleshooting

**SD access fails intermittently:**
- Check 1kΩ resistor connections
- Verify Pin 4 has proper pull-up
- Enable debug logging to see interrupt activity

**Interrupt not triggering:**
- Verify ESP_SD_CS_SENSE is correctly defined
- Check printer's SS line is connected to Pin 4
- Ensure SD_DEVICE_CONNECTION is set to ESP_SHARED_SD

**False positives:**
- Increase pull-up resistor value if needed
- Check for electrical noise on sense line
- Verify ground connection between ESP and printer
