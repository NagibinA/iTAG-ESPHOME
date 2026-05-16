# iTAG BLE Tracker для Home Assistant

[![GitHub Release][releases-shield]][releases]
[![License][license-shield]](LICENSE)
[![ESPHome][esphome-shield]][esphome]

Полноценная интеграция для отслеживания BLE-устройств iTAG через ESP32 с поддержкой всех функций.

## Возможности

- 📍 **Отслеживание присутствия** - определение дома/не дома
- 🔘 **Кнопка** - мгновенная реакция на нажатие
- 🔋 **Батарея** - реальный уровень заряда (без заглушек)
- 📶 **RSSI** - уровень сигнала
- 🔊 **Звук** - функция "Найти брелок"
- 🔄 **Два режима работы** - активный и пассивный

## Принцип работы

ESP32 подключается к iTAG через BLE и передает все данные в Home Assistant через ESPHome.

| Функция | Активный режим | Пассивный режим |
|---------|---------------|-----------------|
| Присутствие | ✅ | ✅ |
| RSSI | ✅ | ✅ |
| Кнопка | ✅ | ❌ |
| Батарея | ✅ | ❌ |
| Найти брелок | ✅ | ❌ |

## Установка

### 1. Прошивка ESP32

1. Установите аддон **ESPHome** в Home Assistant
2. Создайте новое устройство
3. Скопируйте конфигурацию ниже
4. Замените MAC-адрес на свой
5. Скомпилируйте и залейте прошивку

```yaml
esphome:
  name: itag-tracker
  friendly_name: iTAG Tracker

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

logger:

api:
  encryption:
    key: "ВАШ_КЛЮЧ"

ota:
  password: "ВАШ_ПАРОЛЬ"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

esp32_ble_tracker:
  on_ble_advertise:
    then:
      - lambda: |-
          if (x.address_str() == "5B:86:CF:15:45:BA") {
            id(itag_last_seen) = millis();
            id(itag_passive_rssi).publish_state(x.get_rssi());
          }

ble_client:
  - mac_address: "5B:86:CF:15:45:BA"
    id: itag_device
    auto_connect: false
    on_connect:
      then:
        - binary_sensor.template.publish:
            id: itag_presence
            state: ON
    on_disconnect:
      then:
        - binary_sensor.template.publish:
            id: itag_presence
            state: OFF

globals:
  - id: itag_last_seen
    type: int
    restore_value: no
    initial_value: '0'

binary_sensor:
  - platform: template
    id: itag_button
    name: "Кнопка iTAG"
    device_class: power
    filters:
      - delayed_off: 200ms

  - platform: template
    id: itag_presence
    name: "iTAG Presence"
    device_class: presence
    lambda: |-
      if (id(itag_active_mode).state && id(itag_device).connected()) {
        return true;
      }
      if (id(itag_last_seen) > 0 && (millis() - id(itag_last_seen)) < 60000) {
        return true;
      }
      return false;

switch:
  - platform: template
    name: "Активный режим iTAG"
    id: itag_active_mode
    icon: "mdi:bluetooth"
    optimistic: true
    turn_on_action:
      - ble_client.connect: itag_device
    turn_off_action:
      - ble_client.disconnect: itag_device

  - platform: template
    name: "Найти брелок"
    id: find_itag
    icon: "mdi:map-marker"
    turn_on_action:
      - if:
          condition:
            - switch.is_on: itag_active_mode
          then:
            - ble_client.ble_write:
                id: itag_device
                service_uuid: '1802'
                characteristic_uuid: '2a06'
                value: [0x01]
            - delay: 500ms
            - ble_client.ble_write:
                id: itag_device
                service_uuid: '1802'
                characteristic_uuid: '2a06'
                value: [0x00]
          else:
            - lambda: |-
                ESP_LOGW("iTAG", "Активный режим выключен");

sensor:
  - platform: template
    name: "iTAG RSSI"
    id: itag_passive_rssi
    unit_of_measurement: "dBm"
    icon: "mdi:signal"
    update_interval: 30s
    lambda: |-
      if (id(itag_last_seen) > 0 && (millis() - id(itag_last_seen)) < 60000) {
        return id(itag_passive_rssi).state;
      }
      return -120;

  - platform: ble_client
    type: characteristic
    ble_client_id: itag_device
    name: "Button_flag"
    internal: true
    service_uuid: 'ffe0'
    characteristic_uuid: 'ffe1'
    notify: true
    update_interval: never
    on_notify:
      then:
        - binary_sensor.template.publish:
            id: itag_button
            state: ON
        - binary_sensor.template.publish:
            id: itag_button
            state: OFF

  - platform: ble_client
    type: characteristic
    ble_client_id: itag_device
    name: "iTAG Battery"
    service_uuid: '180f'
    characteristic_uuid: '2a19'
    icon: 'mdi:battery'
    unit_of_measurement: '%'
    device_class: battery
    notify: true
    update_interval: never
    on_notify:
      then:
        - lambda: |-
            ESP_LOGD("iTAG", "Battery: %d%%", x);