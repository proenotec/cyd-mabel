# Guía de Hardware - Pantallas CYD

Este documento detalla las diferencias entre las dos versiones de pantallas CYD soportadas por este proyecto.

## 🖥️ Modelos Soportados

### Modelo 1: CYD Capacitiva (WT32-SC01 PLUS) - **RECOMENDADA**
- **Archivo**: `modules/hardware/JC2432W328_landscape.yaml`
- **Touch**: CST816 (Capacitivo I2C)
- **Ventajas**: Touch más preciso, responsivo y suave
- **Precio**: Ligeramente más cara
- **Estado**: ✅ **Configuración por defecto**

### Modelo 2: CYD Resistiva (ESP32-2432S028R)
- **Archivo**: `modules/hardware/2432S028R_landscape.yaml`
- **Touch**: XPT2046 (Resistivo SPI)
- **Ventajas**: Más económica, funciona con stylus
- **Precio**: Más barata
- **Estado**: Requiere cambio de configuración

## 🔧 Diferencias Técnicas

### Comparación de Hardware

| Componente | Capacitiva (WT32-SC01 PLUS) | Resistiva (ESP32-2432S028R) |
|------------|----------------------------|------------------------------|
| **MCU** | ESP32-S3 | ESP32 (WROOM) |
| **Display** | ILI9342 3.5" 320x480 | ILI9342 3.5" 320x480 |
| **Touch Type** | Capacitivo | Resistivo |
| **Touch Driver** | CST816 (I2C) | XPT2046 (SPI) |
| **Backlight Pin** | GPIO27 | GPIO21 |
| **Touch Reset** | GPIO25 | - |
| **Touch CS** | - | GPIO33 |
| **Touch Interrupt** | GPIO36 (opcional) | GPIO36 |
| **SPI Buses** | 1 (compartido) | 2 (separados) |

### Diferencias en Pines

#### Capacitiva (WT32-SC01 PLUS)
```yaml
# Display SPI
spi:
  - id: spi_tft
    clk_pin: GPIO14
    mosi_pin: GPIO13
    miso_pin: GPIO12

# Touch I2C
i2c:
  sda: GPIO33
  scl: GPIO32

# Touchscreen
touchscreen:
  platform: cst816
  reset_pin: GPIO25

# Backlight
backlight_pwm:
  pin: GPIO27
```

#### Resistiva (ESP32-2432S028R)
```yaml
# Display SPI
spi:
  - id: spi_tft
    clk_pin: GPIO14
    mosi_pin: GPIO13
    miso_pin: GPIO12

# Touch SPI (separado)
  - id: touch_screen
    clk_pin: GPIO25
    mosi_pin: GPIO32
    miso_pin: GPIO39

# Touchscreen
touchscreen:
  platform: xpt2046
  spi_id: touch_screen
  cs_pin: GPIO33
  interrupt_pin: GPIO36

# Backlight
backlight_pwm:
  pin: GPIO21
```

## 📝 Cómo Cambiar entre Versiones

### Opción 1: Editar el archivo principal (RECOMENDADO)

Editar `cyd-negro-lvgl-thermostats.yaml`, buscar la sección **"packages:"** y la subsección **"# Hardware configuration"** (aprox. línea 215):

**Para Capacitiva (por defecto)**:
```yaml
# OPCIÓN A: Pantalla CAPACITIVA (WT32-SC01 PLUS) - ✅ RECOMENDADA
hardware: !include modules/hardware/JC2432W328_landscape.yaml

# OPCIÓN B: Pantalla RESISTIVA (comentada)
# hardware: !include modules/hardware/2432S028R_landscape.yaml
```

**Para Resistiva**:
```yaml
# OPCIÓN A: Pantalla CAPACITIVA (comentada)
# hardware: !include modules/hardware/JC2432W328_landscape.yaml

# OPCIÓN B: Pantalla RESISTIVA (activa)
hardware: !include modules/hardware/2432S028R_landscape.yaml
```

**IMPORTANTE**: Solo debes descomentar UNA línea `hardware: !include ...`

### Opción 2: Duplicar configuración (para múltiples dispositivos)

Si tienes ambas pantallas, crea dos archivos:

**cyd-capacitiva.yaml**:
```yaml
substitutions:
  device_name: 'cyd-capacitiva-salon'
  friendly_name: 'CYD Capacitiva Salon'
  hardware_file: modules/hardware/JC2432W328_landscape.yaml
  hardware_type: "Capacitive"

packages:
  base: !include cyd-negro-lvgl-thermostats.yaml
```

**cyd-resistiva.yaml**:
```yaml
substitutions:
  device_name: 'cyd-resistiva-cocina'
  friendly_name: 'CYD Resistiva Cocina'
  hardware_file: modules/hardware/2432S028R_landscape.yaml
  hardware_type: "Resistive"

packages:
  base: !include cyd-negro-lvgl-thermostats.yaml
```

## 🧪 Verificación de Hardware

### Test de Touch Capacitivo
Si el touch no responde con la configuración capacitiva:
1. Verificar conexión I2C (SDA/SCL)
2. Revisar pin de reset GPIO25
3. Probar reducir `update_interval` a 30ms

### Test de Touch Resistivo
Si el touch no responde con la configuración resistiva:
1. Verificar SPI separado para touch
2. Ajustar `threshold` (valor típico: 300-500)
3. Recalibrar coordenadas si es necesario

### Logs de Diagnóstico

Añadir al YAML temporal para diagnóstico:

**Capacitiva**:
```yaml
i2c:
  sda: GPIO33
  scl: GPIO32
  scan: true  # Mostrará dispositivos I2C detectados
```

**Resistiva**:
```yaml
touchscreen:
  on_touch:
    - lambda: |-
        ESP_LOGI("touch", "x=%d, y=%d, x_raw=%d, y_raw=%d",
            touch.x, touch.y, touch.x_raw, touch.y_raw);
```

## 🛒 Dónde Comprar

### Capacitiva (WT32-SC01 PLUS)
- AliExpress: Buscar "WT32-SC01 PLUS"
- Amazon: "ESP32 3.5 inch capacitive touch"
- Precio aprox: $15-25 USD

### Resistiva (ESP32-2432S028R)
- AliExpress: Buscar "ESP32-2432S028R"
- También conocida como "Cheap Yellow Display"
- Precio aprox: $10-15 USD

## 🔍 Identificar tu Pantalla

### Visualmente
- **Capacitiva**: Touch responde sin presión, deslizamiento suave
- **Resistiva**: Requiere presión leve, superficie ligeramente flexible

### Por Código (revisar etiqueta en PCB)
- **Capacitiva**: WT32-SC01 PLUS, JC2432W328, ZX3D50CE08S
- **Resistiva**: ESP32-2432S028R, CYD

### Por Chipset Touch
Conectar USB y revisar logs de ESPHome durante boot:
- **Capacitiva**: Detectará I2C device en 0x15 (CST816)
- **Resistiva**: No mostrará dispositivos I2C

## 📚 Referencias

- [ESPHome ILI9xxx Display](https://esphome.io/components/display/ili9xxx.html)
- [ESPHome XPT2046 Touch (Resistive)](https://esphome.io/components/touchscreen/xpt2046.html)
- [ESPHome CST816 Touch (Capacitive)](https://esphome.io/components/touchscreen/cst816.html)
- [ESP32 Pinout Reference](https://randomnerdtutorials.com/esp32-pinout-reference-gpios/)

---

**Nota**: Este proyecto usa por defecto la versión **Capacitiva** porque ofrece mejor experiencia de usuario para una interfaz táctil LVGL.
