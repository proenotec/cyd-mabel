# ESPHome Climate Control Touchscreen

Sistema de control de clima con pantalla táctil para ESP32-S3 con display LVGL.

## 📋 Tabla de Contenidos

- [Características](#características)
- [Hardware](#hardware)
- [Configuración Rápida](#configuración-rápida)
- [Cómo Añadir/Quitar Termostatos](#cómo-añadirquitar-termostatos)
- [Cómo Añadir/Quitar Persianas](#cómo-añadirquitar-persianas)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Compilación y Carga](#compilación-y-carga)

## ✨ Características

- **Control de múltiples termostatos**: Sistema configurable para N termostatos
- **Control de persianas motorizadas**: Sistema de pares dinámicos para Z persianas
- **Control de luces**: Página dedicada con grid de botones
- **Pantalla de información**: Muestra diagnósticos del dispositivo
- **Notificaciones de texto**: Desde Home Assistant con auto-limpieza
- **Modo idle inteligente**: Cicla automáticamente entre termostatos y apaga pantalla
- **UI LVGL moderna**: Interfaz gráfica fluida con medidores, arcos y botones

## 🔧 Hardware

- **Modelo**: WT32-SC01 PLUS
- **MCU**: ESP32-S3 (ESP32-WROVER-B)
- **Display**: 3.5" 320x480 LCD (ST7796UI driver)
- **Touch**: Capacitivo FT6336U I2C
- **Memoria**: 4MB Flash + 8MB PSRAM

## ⚡ Configuración Rápida

### 1. Identificación del Dispositivo

Editar en `cyd-negro-lvgl-thermostats.yaml`:

```yaml
# SECCIÓN 1: IDENTIFICACIÓN DEL DISPOSITIVO
device_name: 'tu-dispositivo-nombre'
friendly_name: 'Tu Dispositivo Nombre'
```

### 2. Credenciales WiFi y API

Crear/editar `secrets.yaml`:

```yaml
wifi_ssid_gvf: "TuWiFi"
wifi_password_gvf: "TuPassword"
fallback_password: "FallbackPassword"
```

Generar nueva clave API (opcional):

```bash
openssl rand -base64 32
```

## 📝 Cómo Añadir/Quitar Termostatos

### Paso 1: Definir el Termostato en `substitutions`

Editar **SECCIÓN 3** del archivo `cyd-negro-lvgl-thermostats.yaml`:

```yaml
# TERMOSTATO 4 (nuevo ejemplo)
climate_entity_office: climate.office
climate_label_office: "OFFICE"
```

### Paso 2: Añadir Sensores de Modo HVAC

Buscar la sección `text_sensor:` (aprox. línea 520) y añadir:

```yaml
# Climate 4: Office - Mode
- platform: homeassistant
  entity_id: ${climate_entity_office}
  id: hvac_mode_office
  internal: true

# Climate 4: Office - HVAC Action
- platform: homeassistant
  entity_id: ${climate_entity_office}
  id: hvac_action_office
  attribute: hvac_action
  internal: true
  on_value:
    - script.execute: update_climate_display
```

### Paso 3: Añadir Sensores de Temperatura

Buscar la sección `sensor:` (aprox. línea 690) y añadir:

```yaml
# Office sensors
- platform: homeassistant
  entity_id: ${climate_entity_office}
  id: set_temperature_office
  attribute: temperature
  internal: true

- platform: homeassistant
  entity_id: ${climate_entity_office}
  id: current_temperature_office
  attribute: current_temperature
  internal: true
```

### Paso 4: Actualizar `globals`

Buscar `current_climate_entity` (aprox. línea 390) y cambiar el valor inicial si quieres que el nuevo termostato sea el predeterminado:

```yaml
- id: current_climate_entity
  type: std::string
  restore_value: false
  initial_value: '"${climate_entity_office}"'  # Si quieres Office por defecto
```

### Paso 5: Actualizar Script `cycle_climate`

Buscar el script `cycle_climate` (aprox. línea 1228) y actualizar:

```yaml
static const char* climate_entities[] = {
  "${climate_entity_salon}",      // TERMOSTATO 0
  "${climate_entity_bathroom}",   // TERMOSTATO 1
  "${climate_entity_bedroom}",    // TERMOSTATO 2
  "${climate_entity_pasillo}",    // TERMOSTATO 3
  "${climate_entity_office}"      // TERMOSTATO 4 ← NUEVO
};
static const char* climate_labels[] = {
  "${climate_label_salon}",       // TERMOSTATO 0
  "${climate_label_bathroom}",    // TERMOSTATO 1
  "${climate_label_bedroom}",     // TERMOSTATO 2
  "${climate_label_pasillo}",     // TERMOSTATO 3
  "${climate_label_office}"       // TERMOSTATO 4 ← NUEVO
};

const int num_climates = 5;  // ← CAMBIAR de 4 a 5
```

### Paso 6: Actualizar Script `cycle_texto_page_climates`

Buscar el script `cycle_texto_page_climates` (aprox. línea 864) y:

1. Actualizar el array de labels:

```yaml
static const char* climate_labels[] = {
  "${climate_label_salon}",     // 0: Termostato Salon
  "${climate_label_bathroom}",  // 1: Termostato Bathroom
  "${climate_label_bedroom}",   // 2: Termostato Bedroom
  "${climate_label_pasillo}",   // 3: Termostato Pasillo
  "${climate_label_office}",    // 4: Termostato Office ← NUEVO
  "ACS",                        // 5: Sistema ACS (fijo)
  "OTHER",                      // 6: Otra info (fijo)
  "OTHER2"                      // 7: Otra info 2 (fijo)
};
```

2. Añadir un case en el switch (después del case 3):

```yaml
case 4: // TERMOSTATO 4: Office
  current_temp = id(current_temperature_office).state;
  set_temp = id(set_temperature_office).state;
  hvac_action = id(hvac_action_office).state;
  window_state = id(office_window).state;  // ← Ajustar según tu sensor
  break;
```

3. Actualizar el número final (buscar `% 6` y cambiar):

```yaml
id(texto_page_climate_index) = (idx + 1) % 7;  // Era 6, ahora 7 (5 termostatos + 2 extras)
```

### Paso 7: Actualizar Script `update_climate_display`

Buscar el script `update_climate_display` (aprox. línea 1161) y añadir case:

```yaml
case 4: // TERMOSTATO 4: Office
  hvac_mode = id(hvac_mode_office).state;
  hvac_action = id(hvac_action_office).state;
  set_temp = id(set_temperature_office).state;
  current_temp = id(current_temperature_office).state;
  ESP_LOGD("update_climate", "Office - mode='%s', action='%s', has_state=%d",
           hvac_mode.c_str(), hvac_action.c_str(), id(hvac_action_office).has_state());
  break;
```

### Resumen de Líneas a Modificar (para 1 termostato nuevo)

| Sección | Línea Aprox. | Qué Añadir |
|---------|--------------|------------|
| Substitutions | 70-77 | 2 líneas (entity + label) |
| text_sensor | 520-580 | 2 bloques (mode + action) |
| sensor | 690-740 | 2 bloques (set_temp + current_temp) |
| script: cycle_climate | 1236-1249 | 1 línea en cada array + cambiar num_climates |
| script: cycle_texto_page_climates | 875-932 | 1 línea en array + 1 case + cambiar % |
| script: update_climate_display | 1175-1207 | 1 case nuevo |

---

## 🪟 Cómo Añadir/Quitar Persianas

Las persianas se organizan en **PARES** (izquierda + derecha) que se van ciclando automáticamente en la pantalla.

### Paso 1: Definir el Par en `substitutions`

Editar **SECCIÓN 4** del archivo `cyd-negro-lvgl-thermostats.yaml`:

```yaml
# PAR 3 - Izquierda (nuevo)
cover_garage: cover.zbblind7_garage
cover_garage_label: "GARAGE"

# PAR 3 - Derecha (nuevo)
cover_terrace: cover.zbblind8_terrace
cover_terrace_label: "TERRACE"
```

### Paso 2: Añadir Sensores de Estado

Buscar la sección `packages:` (aprox. línea 164) y añadir:

```yaml
# Cover state monitoring for PAR 3
cover_garage_state: !include
  file: modules/sensors/cover_button_state.yaml
  vars:
    uid: garage
    entity_id: ${cover_garage}
    status_label_id: left_cover_status

cover_terrace_state: !include
  file: modules/sensors/cover_button_state.yaml
  vars:
    uid: terrace
    entity_id: ${cover_terrace}
    status_label_id: right_cover_status
```

### Paso 3: Actualizar Arrays en `globals`

Buscar los arrays de covers (aprox. línea 426) y actualizar:

```yaml
- id: left_covers_array
  type: const char*[4]  # ← CAMBIAR [3] a [4]
  restore_value: false
  initial_value: '{
    "${cover_kitchen}",
    "${cover_main_bedroom}",
    "${cover_first_bedroom}",
    "${cover_garage}"  # ← NUEVO
  }'

- id: right_covers_array
  type: const char*[4]  # ← CAMBIAR [3] a [4]
  restore_value: false
  initial_value: '{
    "${cover_salon}",
    "${cover_julia_bedroom}",
    "${cover_bathroom}",
    "${cover_terrace}"  # ← NUEVO
  }'

- id: left_cover_labels
  type: const char*[4]  # ← CAMBIAR [3] a [4]
  restore_value: false
  initial_value: '{
    "${cover_kitchen_label}",
    "${cover_main_bedroom_label}",
    "${cover_first_bedroom_label}",
    "${cover_garage_label}"  # ← NUEVO
  }'

- id: right_cover_labels
  type: const char*[4]  # ← CAMBIAR [3] a [4]
  restore_value: false
  initial_value: '{
    "${cover_salon_label}",
    "${cover_julia_bedroom_label}",
    "${cover_bathroom_label}",
    "${cover_terrace_label}"  # ← NUEVO
  }'
```

### Paso 4: Actualizar Script `cycle_covers_page`

Buscar el script `cycle_covers_page` (aprox. línea 861) y cambiar:

```yaml
id(current_cover_pair_index) = (idx + 1) % 4;  // ← CAMBIAR "% 3" a "% 4"
```

### Resumen de Líneas a Modificar (para 1 par nuevo)

| Sección | Línea Aprox. | Qué Añadir |
|---------|--------------|------------|
| Substitutions | 114-123 | 4 líneas (2 entities + 2 labels) |
| packages: | 164-205 | 2 bloques (estados de cada persiana) |
| globals: arrays | 426-460 | Cambiar [3] a [4] + añadir 1 línea en cada array |
| script: cycle_covers_page | 861 | Cambiar "% 3" al nuevo número |

---

## 📁 Estructura del Proyecto

```
.
├── cyd-negro-lvgl-thermostats.yaml  # Configuración principal (TODO AQUÍ)
├── secrets.yaml                     # Credenciales WiFi/API (crear manualmente)
├── CLAUDE.md                         # Documentación del proyecto
├── README.md                         # Este archivo
└── modules/                          # Módulos reutilizables
    ├── base/                         # WiFi, tiempo, colores, fuentes
    ├── hardware/                     # Configuraciones de pantalla
    ├── buttons/                      # Templates de botones
    ├── sensors/                      # Monitores de estado HA
    └── screens/                      # Pantallas de arranque
```

## 🚀 Compilación y Carga

### Validar configuración

```bash
esphome config cyd-negro-lvgl-thermostats.yaml
```

### Compilar

```bash
esphome compile cyd-negro-lvgl-thermostats.yaml
```

### Cargar vía WiFi

```bash
esphome upload cyd-negro-lvgl-thermostats.yaml
```

### Cargar vía USB (primera vez)

```bash
esphome upload --device /dev/ttyUSB0 cyd-negro-lvgl-thermostats.yaml
```

### Ver logs

```bash
esphome logs cyd-negro-lvgl-thermostats.yaml
```

## 🔍 Troubleshooting

### Error: "entity_id not found"

Verificar que la entidad existe en Home Assistant con el nombre exacto.

### Pantalla en blanco

1. Verificar hardware del display (substitutions: hardware)
2. Revisar logs: `esphome logs ...`
3. Comprobar que LVGL buffer_size está en 10%

### Termostato no aparece

Verificar que:
1. Está en los arrays de `cycle_climate`
2. Tiene sensores de modo, acción y temperaturas
3. El número `num_climates` es correcto

### Persianas no ciclan

Verificar:
1. Arrays tienen el mismo número de elementos
2. El `% N` en `cycle_covers_page` es correcto
3. Los IDs de las etiquetas de estado coinciden

## 📄 Licencia

Este proyecto está disponible bajo licencia MIT.

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor, crea un issue antes de hacer cambios grandes.

---

**Generado con [Claude Code](https://claude.com/claude-code)**
