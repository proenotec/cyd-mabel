# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESPHome configuration for a climate control touchscreen display running on ESP32 hardware (specifically WT32-SC01 PLUS). The device provides a custom LVGL-based UI for controlling Home Assistant climate entities, lights, and motorized blinds/covers.

**Hardware**: ESP32-S3 with 3.5" 320x480 touchscreen (ST7796UI driver, FT6336U capacitive touch)

**Version**: v2.0.0 - Reorganized for easy configuration with flexible number of thermostats and covers

**Hardware Support**: Capacitive (WT32-SC01 PLUS) and Resistive (ESP32-2432S028R) CYD displays

## 📖 Configuration Organization (v2.0.0)

The configuration file has been reorganized for maximum clarity and ease of modification:

### Structure of Main Configuration File

All user-configurable settings are at the **TOP** of `cyd-negro-lvgl-thermostats.yaml` in clearly marked sections:

1. **SECCIÓN 1**: Device identification (name, friendly_name)
2. **SECCIÓN 2**: Security and credentials (WiFi, API key, OTA password)
3. **SECCIÓN 3**: Thermostat configuration (supports N thermostats)
4. **SECCIÓN 4**: Cover/blind configuration (supports Z covers organized in pairs)
5. **SECCIÓN 5**: Hardware selection (Capacitive vs Resistive display)
6. **SECCIÓN 6**: General system settings (logging level, etc.)

### Adding/Removing Thermostats

The system now supports a flexible number of thermostats. Each section that needs updating has **inline comments** indicating:
- What needs to be changed
- Which line numbers to look at
- Examples of what to add

**Key sections to update** (all clearly marked with `# IMPORTANTE:` comments):
- Line ~55-77: Define entities in `substitutions`
- Line ~520-580: Add `text_sensor` for HVAC mode/action
- Line ~690-740: Add `sensor` for temperatures
- Line ~1228-1249: Update arrays in `cycle_climate` script
- Line ~864-932: Update `cycle_texto_page_climates` script
- Line ~1161-1207: Update `update_climate_display` script

### Adding/Removing Covers

Covers are organized in **PAIRS** (left + right) that cycle automatically:
- Line ~94-123: Define cover entities and labels in `substitutions`
- Line ~164-205: Add cover state sensors in `packages`
- Line ~426-460: Update 4 arrays in `globals` (entities and labels for left/right)
- Line ~861: Change `% 3` to number of pairs

### Hardware Selection (Capacitive vs Resistive)

The project supports **two versions** of CYD displays:
- **Capacitive (Default)**: WT32-SC01 PLUS with CST816 I2C touch - Better UX
- **Resistive**: ESP32-2432S028R with XPT2046 SPI touch - More economical

**To switch hardware**:
- Line ~215-226: In `packages:` section under "Hardware configuration"
- Comment/uncomment the appropriate `hardware: !include ...` line
- Only ONE hardware line should be uncommented at a time
- See `HARDWARE.md` for detailed comparison and identification guide

### Visual Indicators in Code

All critical sections are marked with:
```yaml
# ═══════════════════════════════════════════════════════════════════════════
# Section Title with Visual Separator
# ═══════════════════════════════════════════════════════════════════════════
# IMPORTANTE: Instructions on what to update
```

**→** Use symbols like `←` to indicate values that commonly need changing:
```yaml
const int num_climates = 4;  // ← CAMBIAR al número total de termostatos
```

## 📚 Documentation Files

- **README.md**: User-facing documentation with step-by-step guides
- **CLAUDE.md**: This file - developer/AI assistant guidance
- **HARDWARE.md**: Detailed hardware comparison and selection guide
- **cyd-negro-lvgl-thermostats.yaml**: Main config with inline documentation

## Build and Development Commands

### Compile and Upload
```bash
# Compile the configuration (validates YAML)
esphome compile cyd-negro-lvgl-thermostats.yaml

# Upload to device over WiFi (after initial flash)
esphome upload cyd-negro-lvgl-thermostats.yaml

# Upload over USB (initial flash or when WiFi unavailable)
esphome upload --device /dev/ttyUSB0 cyd-negro-lvgl-thermostats.yaml

# View logs from device
esphome logs cyd-negro-lvgl-thermostats.yaml

# Clean build artifacts
esphome clean cyd-negro-lvgl-thermostats.yaml
```

### Validation and Testing
```bash
# Validate YAML syntax and configuration
esphome config cyd-negro-lvgl-thermostats.yaml

# Compile without uploading (useful for CI/testing)
esphome compile cyd-negro-lvgl-thermostats.yaml
```

## Architecture

### Modular Package System

The project uses ESPHome's `packages` feature extensively. The main configuration file (`cyd-negro-lvgl-thermostats.yaml`) imports reusable modules from the `modules/` directory:

- **modules/base/**: Core functionality (WiFi, logging, time, colors, fonts, touchscreen)
- **modules/hardware/**: Hardware-specific configurations for different display models
- **modules/buttons/**: Reusable button templates for lights and switches
- **modules/sensors/**: State monitoring components for Home Assistant entities
- **modules/screens/**: Boot screen definitions for various screen sizes

### Variable Substitution Pattern

The codebase heavily uses templated YAML with variable substitution (`vars:`). When including a module, you can pass parameters:

```yaml
- button: !include
    file: modules/buttons/light_button.yaml
    vars:
      uid: 1
      x: 20
      y: 60
      entity_id: "light.petinfan_2_fanlight_2"
```

The included file references these variables using `${var_name}` syntax.

### Page Architecture

The UI has 5 main pages implemented in LVGL:

1. **main_page**: Climate control with thermostat dial and temperature display
2. **texto_page**: Text notification display with scrolling content
3. **lights_page**: 2x2 grid of light control buttons
4. **covers_page**: Motorized blind controls with cycling pairs (6 total covers shown 2 at a time)
5. **info_page**: Device diagnostics (reuses boot screen with additional buttons)

### State Management

- **Global variables** (`globals:`) track UI state (current page, climate index, cover pair index, idle mode)
- **Scripts** (`script:`) handle page transitions, climate cycling, and idle behavior
- **Home Assistant sensors** (`platform: homeassistant`) provide real-time entity state
- **Binary sensors** monitor entity states and trigger UI updates via `on_state:` handlers

### Idle Mode System

The device implements a sophisticated idle behavior:

1. After 30 seconds on non-main pages → auto-return to main_page
2. On main_page → auto-cycle through 4 climate zones every 5 seconds
3. After screen timeout (configurable) → dim backlight in stages then turn off
4. Touch or Home Assistant connection → wake up and restore

Key scripts: `start_idle_mode`, `stop_idle_mode`, `idle_cycle_thermostats`, `idle_page_timeout`

## Key Design Patterns

### Dynamic Cover Cycling

The covers_page displays 2 covers at a time from 6 total covers. The system cycles through 3 pairs automatically:
- Pair 0: Kitchen + Salon
- Pair 1: Main Bedroom + Julia's Bedroom
- Pair 2: First Bedroom + Bathroom

Arrays defined in `globals:` store cover entities and labels. The cycling logic uses `current_cover_pair_index` and adjusts delay based on recent button presses (5s default, 30s after interaction).

### State Synchronization

Each interactive button has a corresponding state sensor in `modules/sensors/`:
- `switch_or_light_button_state.yaml`: Monitors light/switch state and updates button colors
- `cover_button_state.yaml`: Monitors cover position and updates status indicators

The pattern: Create a `binary_sensor` with `platform: homeassistant`, then use `on_state:` to trigger `lvgl.widget.update` actions.

### Climate Control Flow

1. User selects climate zone via up/down buttons (or auto-cycles in idle mode)
2. `cycle_climate` script updates `current_climate_index` and `current_climate_entity`
3. `update_climate_display` script reads temperature sensors and HVAC state for active zone
4. `set_hvac_action` script updates UI elements (power button color, heating icon, meter display)
5. Temperature changes via knob or buttons call `homeassistant.service: climate.set_temperature`

## Important Configuration Details

### Customization Points (Top of Main File)

All user-specific settings are in the `substitutions:` section:
- `device_name` and `friendly_name`: Device identifiers
- WiFi credentials (from secrets.yaml)
- Climate entity IDs for 4 zones (salon, bathroom, bedroom, pasillo)
- Cover entity IDs for 6 motorized blinds
- API encryption key (unique per device)
- MDI icon codes and color definitions

### Hardware Variants

The project supports both capacitive and resistive touchscreens:
- **Capacitive** (current): `modules/hardware/JC2432W328_landscape.yaml`
- **Resistive** (commented): `modules/hardware/2432S028R_landscape.yaml`

These files define GPIO pins, SPI configuration, display driver settings, and touchscreen calibration.

### Boot Sequence

1. Device boots → `on_boot:` script plays beep and shows boot screen for 60s
2. Boot screen displays ESPHome logo + diagnostics (WiFi, IP, uptime)
3. After delay → `hide_boot_screen` reveals main_page and starts idle mode
4. `system_ready` global flag prevents premature text notifications during startup

## Working with This Codebase

### Adding a New Light Button

1. Add entity to lights_page in main YAML:
```yaml
- button: !include
    file: modules/buttons/light_button.yaml
    vars:
      uid: 5  # unique ID
      x: 20
      y: 220
      entity_id: "light.new_light"
```

2. Add corresponding state sensor:
```yaml
button_5_state: !include
  file: modules/sensors/switch_or_light_button_state.yaml
  vars:
    uid: 5
    entity_id: "light.new_light"
```

### Modifying HVAC Display

Temperature ranges are defined in the meter scale (lines 1566-1567):
- `range_from: 160` (16°C × 10)
- `range_to: 340` (34°C × 10)

Values are multiplied by 10 for decimal precision with integer display.

### Debugging UI Issues

- Set `log_level: "DEBUG"` in substitutions (default is DEBUG)
- LVGL log level is WARN to reduce noise (line 1372)
- Component-specific logging in `logger:` section (line 13-22)
- Use ESP logs to trace script execution and state changes

### Text Notifications System

Home Assistant sends notifications via `input_text.mi_texto1`:
1. Text appears on texto_page with scrollable container
2. Climate info cycles at bottom every 5s
3. "BACK" button sends clear request to HA via `clear_texto_command` sensor
4. HA automation must clear the input_text to prevent re-display
5. `clearing_texto` flag prevents race conditions during clear operation

## ESPHome-Specific Notes

- This project uses `esp-idf` framework (not Arduino) for better ESP32-S3 support
- LVGL buffer set to 10% for performance/memory balance
- Display uses `auto_clear_enabled: false` and `update_interval: never` (LVGL manages refresh)
- Preferences saved to flash every 1 minute to preserve settings
- API encryption required for Home Assistant connection

## File Organization

```
.
├── cyd-negro-lvgl-thermostats.yaml  # Main configuration
└── modules/
    ├── base/           # WiFi, time, colors, fonts, core functionality
    ├── hardware/       # Device-specific pin/driver configs
    ├── buttons/        # Reusable button templates
    ├── sensors/        # HA entity state monitors
    └── screens/        # Boot screens for different displays
```

When modifying, prefer editing module files for reusability. Only add to main YAML for device-specific configuration.
