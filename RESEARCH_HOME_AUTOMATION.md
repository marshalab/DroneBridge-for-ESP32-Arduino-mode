# DroneBridge-for-ESP32 — исследование применимости для домашней автоматики

**Дата:** 2026-10-19  
**База:** `DroneBridge-for-ESP32-Arduino-mode` v2.4.1 (ESP-IDF 5.4.4, chip ESP32/C3/C6/S2/S3), ветка `master` @ `ed1b9ef`  
**Цель:** ответить на заметки — можно ли быстро взять DroneBridge как радиоканал для датчиков: опрос → MQTT, загрузка профилей/скриптов, MQTT → broadcast, своё ядро + обновление по воздуху.

> TL;DR для спешащих — **DroneBridge как готовый “радиомост” очень хорош для 1–2 недель MVP**, но **как полноценная платформа домашней автоматики — потребует форка и переработки**. Быстрее всего завести канал: оставить DroneBridge в режиме `Transparent` (TCP 5760 / UDP 14550 / ESP-NOW) и повесить MQTT-шлюз на одной ноде-GND (малине/ESP32-GND). Нативный MQTT-клиент внутри каждой ноды, файловую систему для скриптов и OTA придётся дописывать. В open-source ветке OTA сознательно отсутствует — OTA есть только в платной DLSE; для open-source придётся ввести OTA-раздел самостоятельно.

---

## 1. Что такое DroneBridge сегодня (факты)

Малая прошивка-мост: `UART/JTAG ↔ Radio`. На ESP32 поднимает либо Wi-Fi AP/STA/LR, либо ESP-NOW LR (connectionless broadcast до ~1 км [6](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/drone-swarm-control)), либо BLE. Данные гоняет прозрачно или с лёгким парсингом MAVLink/MSP/LTM и инжектом `RADIO_STATUS` для GCS.

**Архитектура (упрощённо):**

```
[Датчик/FC] --UART/JTAG--> [ESP32 DroneBridge] --WiFi/ESP-NOW/BLE--> [ESP32 GND] --TCP 5760 / UDP 14550--> [QGC / твой gateway]
                                 |                        |
                                 +-- HTTP 80 /api/* ------+-- web-конфигуратор (SPIFFS /www = 192KB)
                                 +-- NVS settings --------+-- 27 параметров (ssid/pass/channel/proto/pack_size...)
                                 +-- mDNS dronebridge.local
```

- **Режимы** `SYS_ESP32_MODE`: 1 AP, 2 STA, 3 AP-LR, 4 ESP-NOW AIR, 5 ESP-NOW GND, 6 BLE [2](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/configuration)
- **Сериализация:** `SERIAL_TEL_PROTO`: 1 MSP/LTM, 4 MAVLink, 5 Transparent (любой байтовый поток)
- **Сеть:** TCP 5760, UDP 14550, внутренний multicast 232.10.11.12:1606 для телеметрии, HTTP REST.
- **Шифрование:** WPA2 PSK в Wi-Fi, AES-GCM 256 в ESP-NOW (соль `GenSalt147894562`, PBKDF2-SHA256 10k итераций — см. `db_esp_now.c`).
- **Frontend:** `frontend/` собирается в один `index.html` с inline CSS/JS (специально, чтобы веб-сервер не грузил пачку файлов).
- **Партиции:** `partitions.csv` → `factory @ 0x10000 1536K` + `spiffs www 0x30000`. **OTA-партиций нет**. Для OTA придётся менять таблицу.

**Документация:** [Quick Start](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/quick-start) [3](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/quick-start), [Overview](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/untitled) [1](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/untitled), [Configuration](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/configuration) [2](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/configuration).

---

## 2. Разбор твоих пунктов

### 2.1 Опрос датчиков → вывод в MQTT

**Сейчас в DroneBridge опроса нет.** Он не знает про I²C/SPI/1-Wire — он просто перекидывает байты с UART в радио. То есть датчик должен сам что-то писать в UART.

**Как сделать, 3 уровня сложности:**

| Подход | Где живёт опрос | Плюс | Минус |
|---|---|---|---|
| **A. Внешний MCU-датчик + Transparent** | Датчик-плата (ATtiny/STM32/другой ESP) пишет JSON/CBOR в UART DroneBridge | 0 изменений в DroneBridge, работает сегодня | +1 чип, UART-бутылочное горлышко |
| **B. Гибрид: твой таск рядом с DroneBridge** | FreeRTOS-таск в `main.c` / `db_esp32_control.c` читает I²C и вызывает `db_send_to_all_clients()` / `db_send_to_all_radio_clients()` | Одна плата, один радиоканал | Нужен форк, следить за памятью/вотчдогом |
| **C. Нативный MQTT на ноде** | Тот же таск + `esp-mqtt` клиент паблишит прямо в брокер (STA-режим) | Шлюз не нужен, каждая нода — умная | Требует Wi-Fi клиент/ифраструктуру, +30KB RAM, TLS опционально |

**Рекомендация для MVP — A → B.**

- **MVP (1 неделя):** оставь DroneBridge untouched, датчик шлёт в UART строку `{"id":"kitchen","t":22.4,"h":58}\n`. На GND-стороне (ESP32-GND в STA или малина) висит tiny Python/Go шлюз: слушает TCP 5760 / UDP 14550, парсит и паблишит в MQTT (`home/sensors/<id>`). Так делают Swarm-гайды DroneBridge — ESP-NOW ноды стекаются к одной GND-шлюзу [6](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/drone-swarm-control).
- **Следующий шаг — B:** добавь свой сенсор-драйвер как отдельный компонент `components/sensor_hub/`. В `app_main()` после `db_start_control_module()` запускай `xTaskCreate(sensor_task, ...)`. Сенсор-таск с интервалом 1–5с читает BME280/DS18B20 и форматирует. Для публикации используй существующие `db_send_to_all_*` — они уже умеют резать на `DB_ESPNOW_PAYLOAD_MAXSIZE` (~200 байт).

> Важно: `DB_SERIAL_PACK_SIZE_MAX = 1023`, `SERIAL_PACK_SIZE` по умолчанию 576, `serial_timeout` 20мс. Для сенсорного JSON ставь `pack_size=128–256` и `proto=Transparent` — иначе MAVLink-парсер будет ждать валидный фрейм и дропать.

**Вывод в MQTT — два места:**

1. **На GND-шлюзе (рекомендовано):** один MQTT-клиент на всех. Плюс — ноды остаются ultra-low-power, могут жить в ESP-NOW без Wi-Fi инфры.
2. **На каждой ноде (STA):** добавляешь `esp-mqtt` (сейчас в `sdkconfig.defaults` выключен: `CONFIG_MQTT_PROTOCOL_311=n`). Включаешь `CONFIG_MQTT_PROTOCOL_311=y`, `CONFIG_MQTT_TRANSPORT_SSL=y` если нужен TLS, добавляешь в Kconfig. Пример публикации:
```c
esp_mqtt_client_handle_t mqtt = esp_mqtt_client_init(&(esp_mqtt_client_config_t){
  .broker.address.uri = "mqtt://192.168.1.10:1883",
  .credentials.client_id = "db-sensor-kitchen"
});
esp_mqtt_client_start(mqtt);
esp_mqtt_client_publish(mqtt, "home/sensors/kitchen", json, 0, 1, 0);
```
В STA-режиме DroneBridge уже умеет статический IP/DNS — удобно для брокера.

### 2.2 Загрузка настроек, профилей, скриптов в ноду датчика

**Что есть сейчас:**

- 27 параметров в NVS (`db_parameters.h: DB_PARAM_TOTAL_NUM 27`), доступны через `GET/POST /api/settings` (JSON) и через MAVLink `PARAM_*` (только 18 числовых, строковые — только REST).
- Валидация в `db_param_is_valid_*`, персистенс `db_write_settings_to_nvs()` → `esp_restart()`.
- Веб-форма на `/` сохраняет всё разом.

**Чего нет:** произвольных файлов, профилей, скриптов. SPIFFS `www` всего 192KB и забит фронтендом. Нет файлового API.

**Как добавить:**

1. **Для настроек/профилей** — не городить FS, расширяй NVS/JSON. Добавь новый `db_parameter_t` или отдельный неймспейс `nvs_flash` `profiles`. REST-эндпоинт `POST /api/profiles` принимает `{"profile":"night","interval_ms":5000,"threshold":30}` → пишешь в NVS как blob/string. Твой сенсор-таск читает при старте. Это 1–2 дня работы, без изменения партиций.
2. **Для скриптов** — нужен FS. Варианты:
   - Расширить `www` до 0x50000+ и хранить скрипты в `/www/scripts/` рядом с `index.html`. Потребует правки `partitions.csv` и пересборки `spiffs`.
   - Выделить отдельный раздел `storage,data,spiffs,,0x40000` (как в ESPHome). Тогда HTTP хендлеры `GET /api/scripts/<name>` / `POST /api/scripts` — простые `open/read/write`.
3. **Интерпретатор:** для “скриптов” на ESP32 обычно берут Lua (Berry, MicroPython) или свой DSL JSON-правил. Для домашней автоматики чаще хватает JSON-правила: `{"if":{"sensor":"temp","gt":28},"then":{"broadcast":"relay/on"}}`. Парсишь в C без интерпретатора.

Пример REST для профиля (псевдокод):
```c
httpd_uri_t profile_post = {.uri="/api/profiles", .method=HTTP_POST, .handler=profile_post_handler};
cJSON *root = cJSON_Parse(buf);
cJSON *interval = cJSON_GetObjectItem(root,"interval_ms");
if(cJSON_IsNumber(interval)) nvs_set_i32(handle,"interval_ms",interval->valueint);
```

Фронтенд: добавь секцию в `frontend/index.html` + логику в `dronebridge.js` (там уже есть `send_json`/`get_json`).

### 2.3 MQTT → broadcast управление

**Сейчас DroneBridge уже broadcustит**, но наоборот: `UART → TCP/UDP/ESP-NOW`. Обратный путь `GCS → UART` тоже есть (`db_parse_mavlink_from_radio`, `xQueueReceive(db_uart_write_queue)`).

**Хочешь `MQTT → broadcast → ноды`:**

- **Wiring:** `MQTT брокер → GND-шлюз (подписчик) → DroneBridge broadcast → все AIR ноды → UART/реле`.
- **Реализация:**
  1. На GND (STA или AP) запускаешь MQTT-подписчика (Python на малине или `esp-mqtt` на ESP32-GND).
  2. На сообщение в `home/actuators/#` вызываешь `db_send_to_all_clients(tcp, udp, payload, len)` или, в ESP-NOW GND, `db_send_to_all_espnow()`.
  3. На AIR ноде твой сенсор-таск ловит данные из `db_uart_write_queue` (если proto=Transparent — просто `write_to_serial()` приходит извне, но тебе нужно перехватить). Проще — в `control_module_esp_now()` ветка `else // transparent` уже делает `write_to_serial()`. Добавь хук: если payload — твой командный JSON (`{"cmd":"relay","val":1}`), не пиши в UART, а дёргай GPIO.

**Важные нюансы ESP-NOW:**

- В ESP-NOW каждый пакет шифруется и идёт broadcast (`FF:FF:FF:FF:FF:FF`). Лимит `ESP_NOW_MAX_DATA_LEN - IV - TAG - header = ~200 байт` (см. `DB_ESPNOW_PAYLOAD_MAXSIZE`). Большие JSON режутся автоматически в `db_send_to_all_clients()`, но лучше держать команду <200 байт.
- ESP-NOW GND хранит до 19 пиров (`DB_ESPNOW_MAX_BROADCAST_PEERS 19` [db_espnow_limits.h]). Для дома хватит.
- В текущей прошивке AIR→AIR форвард по умолчанию выключен, включается галкой `Forward MAVLink from other AIR` + `DB_PARAM_ESPNOW_AIR2AIR`. Для домашней сетки удобнее star: все AIR шлют в GND, GND ретранслирует. Если нужен mesh AIR↔AIR — включи галку, но помни про шторм.

**Готовая схема MVP без правки прошивки:**

```
[HomeAssistant / Node-RED] --MQTT publish "home/relay/kitchen/set ON"--> [Raspberry Pi GND: mqtt_to_udp.py] --UDP 14550--> [ESP32 AP (DroneBridge)] --WiFi/ESP-NOW--> [ESP32 AIR датчики] --> GPIO relay
```

`mqtt_to_udp.py` 15 строк:
```python
import paho.mqtt.client as mqtt, socket
sock=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
def on_message(c,u,m): sock.sendto(m.payload, ("192.168.2.1",14550))
mqtt.Client().on_message=on_message; mqtt.Client().connect("192.168.1.10"); mqtt.Client().subscribe("home/actuators/#"); mqtt.Client().loop_forever()
```

### 2.4 Скрещивание DroneBridge движка + своё ядро

**Где живёт логика DroneBridge:**

- `app_main()` → `db_init_wifi_*` / `db_start_espnow_module()` → `db_start_control_module()` → два FreeRTOS-пула: Control и ESP-NOW.
- Control крутит `while(1)` с `read_process_serial_link()` + приём TCP/UDP/BLE и очередь `db_uart_write_queue`.
- Все параметры — глобали `DB_PARAM_*`, доступ через `extern`.

**Стратегии скрещивания:**

**Вариант 1 — “Тонкий мост” (рекомендован для старта):** не трогать DroneBridge, свой код — отдельный процесс на GND-шлюзе (малина/второй ESP). DroneBridge остаётся чистым радиомостом. Плюс: 0 риска сломать радио, обновления DroneBridge накатываешь штатно. Минус: +1 железка.

**Вариант 2 — “Hybrid Core” (оптимальный среднесрок):** форк `main/` + свой компонент `components/homeauto_core/`. Свой core — FreeRTOS-таски `sensor_task`, `actuator_task`, `mqtt_task` с очередями. Общение с DroneBridge через уже существующие `db_send_to_all_radio_clients()` и `db_uart_write_queue`. Важно не трогать Wi-Fi init (там `esp_netif_create_default_wifi_ap`, `esp_wifi_init`), а добавить `sensor_task` после. Память: ESP32-C3 ~320KB SRAM, DroneBridge ест ~80KB, остаётся ~100KB под тебя — хватит на BME280 + MQTT без TLS. С TLS бери C6/S3.

**Вариант 3 — “Fork + рефактор”:** вынести DroneBridge в либу `libdronebridge.a` и писать свой `app_main`. Долго, но чисто для продукта. Требует понимания `idf.py set-target` и `Kconfig.projbuild`.

**Практические правила гибрида:**

- Создай ветку от `master`, не от `arena/...` с ограничениями AGENTS.md. AGENTS.md в этой арене запрещает OTA/C5/5.5/DLSE-фичи — для домашней автоматики это ок, но имей в виду.
- Не меняй `CONFIG_LWIP_MAX_SOCKETS=16` и `max_uri_handlers=9` без нужды.
- Сенсор-таски — низкий приоритет (1-2), Control — 5-6, иначе потеряешь пакеты.
- Используй `esp_timer` для опроса, а не `vTaskDelay` в горячем цикле — смотри как сделаны `db_timer_start_mavlink_heartbeat()`.

Пример врезки своей задачи:
```c
// в app_main() после db_start_control_module()
xTaskCreate(sensor_poll_task, "sensor", 4096, NULL, 2, NULL);

_Noreturn void sensor_poll_task(void *a){
  for(;;){
    char buf[128];
    int len = snprintf(buf,sizeof(buf),"{\"t\":%.1f,\"h\":%.1f}\n", read_temp(), read_hum());
    db_send_to_all_radio_clients((uint8_t*)buf, len);
    // или если STA+MQTT:
    // esp_mqtt_client_publish(mqtt, "home/sensors/kitchen", buf, 0, 0, 0);
    vTaskDelay(pdMS_TO_TICKS(5000));
  }
}
```

### 2.5 Обновление прошивки по воздуху — программа и процедура

**Сейчас в open-source OTA нет.** Это сознательное ограничение проекта и прямо прописано в `AGENTS.md` этого репозитория:

> Do not implement OTA Updates.

На сайте и GitHub OTA значится только для **Drone Light Show Edition (DLSE)** — коммерческого форка с WiFi6/5GHz, Skybrush и OTA [1](https://github.com/DroneBridge/ESP32) [9](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/safety-and-integration-guideline):

> DLSE: Over The Air (OTA) Updates for the ESP32 — update firmware on every drone over Wi-Fi without touching USB [14](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/overview-drone-light-show-edition)

**Какие программы существуют:**

1. **Официальный онлайн-флешер (рекомендован для open-source):** https://drone-bridge.com/flasher/ — WebSerial во браузере, выбираешь версию `Stable` + flavour `ESP32(-XX)` / `ESP32-xx Official HW` / `USBSerial` для GND. Работает через USB, без OTA [4](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/installation) [10](https://drone-bridge.com/flasher/).
2. **ESP-IDF локально:** `idf.py set-target esp32c3 && idf.py build flash monitor` (требует USB-UART/JTAG). Единственный способ для кастомных сборок.
3. **DLSE Commercial Support Suite (OTA, только DLSE):** Python-скрипт `batch_ota_update_allinone.py` сканирует подсеть (`--subnetmask 192.168.1.0/24`), находит DLSE-ноды по broadcast-порту, шлёт `.bin` по Wi-Fi, сохраняет настройки/лицензии [9](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/dlse-commercial-support-suite). Требует выключить Skybrush Live (порт занят). Пример:
```bash
python batch_ota_update_allinone.py --release-folder DroneBridge_ESP32DLSE_BETA4 --subnetmask 192.168.1.0/24 --target-version 1.0.0-beta.3
```

**Если OTA нужен в open-source — придётся дописать самому (обычные ESP32-практики):**

- **Партиции:** замени `partitions.csv` на OTA-схему: `factory, ota_0, ota_1, www, nvs`. Пример на 4MB flash:
```
nvs,data,nvs,0x9000,0x6000
otadata,data,ota,0xf000,0x2000
phy_init,data,phy,0x11000,0x1000
factory,app,factory,0x20000,1M
ota_0,app,ota_0,,1M
ota_1,app,ota_1,,1M
www,data,spiffs,,0x30000
```
- **Компонент:** включи `CONFIG_HTTP_SERVER` + `app_update`, добавь HTTP хендлер `POST /api/ota` который пишет в `esp_ota_begin/write/end/set_boot_partition`. Или используй готовый `esp_https_ota` / `ArduinoOTA` (порт 3232, `ArduinoOTA.handle()` в `loop()`).
- **Безопасность:** OTA только по WPA2/STA или локальному AP, обязательно `CONFIG_SECURE_BOOT` + пароль на `/api/ota`. Не катай OTA в ESP-NOW — там нет TCP.
- **Процедура OTA “по-домашнему”:** GND-нода в STA качает `.bin` с твоего сервера, сама перепрошивается по `esp_https_ota`, потом ретранслирует бинарник AIR-нодам по HTTP (AIR должны временно поднять AP/STA). Сложно, поэтому для дома проще: AIR-ноды шьются по USB раз в квартал, а логика обновляется как скрипты-профили через REST без перепрошивки.

**Вывод:** для быстрого старта OTA не делай — используй флешер + профили. Для продукта заложи OTA-партиции сразу, но реализуй после MVP.

---

## 3. Глюки и ограничения, которые ловили пользователи

Из релизов и issues [1](https://github.com/DroneBridge/ESP32/releases) [5](https://githubhelp.com/DroneBridge/ESP32):

- **Рестарт AP не дерегает клиентов** — клиенты думают что связаны, нужен рестарт клиента или оффлайн AP >3с.
- **MAVLink-парсинг теряет ~0.5 пакета/с** — из-за неотвеченных сообщений GCS→ESP32. Для прозрачного режима это не критично, для прозрачного домашнего JSON ставь `Transparent`.
- **Веб-интерфейс открывает новую вкладку при Save** (старые баги), лечится power-cycle после сохранения.
- **TCP/UDP “connection refused”** — часто из-за несовпадения `wifi_pass`/`espnow_secret` или `max packet size` >250 в ESP-NOW.
- **USBSerial flavour** — для флеша нужно зажать BOOT при подаче питания, иначе флешер не видит плату [4](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/installation).
- **LR/ESP-NOW невидим для обычных устройств** — обязательно имей план возврата: короткое нажатие BOOT → AP `dronebridge` / долгое → reset.

Для дома самые болезненные: 19 пиров лимит, 200 байт на пакет, 192KB SPIFFS, отсутствие OTA и MQTT из коробки.

---

## 4. Рекомендация “по-быстрому” (как сделать канал за выходные)

**Цель:** опрос датчиков + MQTT + broadcast управление без форка.

1. **Закажи 2× ESP32-C6 official HW** (лучшая дальность, RF-switch) или любые C3/S3 с USB-UART.
2. **Прошей обе:** флешер → `Stable` → `ESP32-C6` → first boot в AP `dronebridge` → 192.168.2.1 → выставь `Transparent`, `pack_size 128`, `timeout 50ms`, пины TX/RX под свою разводку. Проверь в `WiFi Client`/`AP` (веб будет доступен), потом переведи в `ESP-NOW AIR` + `GND` с одинаковым `espnow_secret` (нажми Rotate на GND, скопируй на AIR).
3. **Датчик-нода (AIR):** ESP32-AIR + BME280 по I²C + реле на GPIO. Прошивка — **не DroneBridge**, а ESPHome/Tasmota/твой Arduino-скетч, который пишет в UART в сторону DroneBridge? Нет — проще: датчик-нода = сам DroneBridge-AIR + твой сенсор-таск (если хочешь 1 чип) **или** две платы: сенсор-Arduino → UART → DroneBridge-AIR (transparent). Для выходных бери 2 платы, чтобы не форкать.
4. **GND-шлюз:** малина/ноут в WiFi STA, подключается к AP DroneBridge *или* ESP32-GND в USBSerial → `mqtt_to_udp.py` (см. выше) + Node-RED → Home Assistant. Подписка `home/actuators/#` → UDP 14550.
5. **Проверка:** `mosquitto_pub -t home/actuators/relay -m '{"id":"kitchen","val":1}'` → реле щёлкает.
6. **Профили:** пока храни в Home Assistant, не в ноде. Когда нужно — добавь `POST /api/settings` с новым `interval_ms`.

**Когда это взлетит — планируй форк с сенсор-таском и FS для профилей.**

---

## 5. План развития (оценка)

| Этап | Что | Срок | Риск |
|---|---|---|---|
| **MVP 0** | 2× ESP32 + Transparent + GND MQTT шлюз на питоне | 1–2 дня | Низкий |
| **MVP 1** | Форк, сенсор-таск I²C, REST `/api/profiles` (NVS) | 1–2 недели | Средний (память) |
| **MVP 2** | FS для скриптов, `MQTT→ESP-NOW` внутри GND, GPIO-актуаторы | 2–3 недели | Средний |
| **Продукт** | OTA-партиции + `POST /api/ota`, TLS MQTT, OTA-реле | 1–2 месяца | Высокий |

---

## 6. Итог — стоит ли DroneBridge для дома?

- **Да, если** нужен дальнобойный прозрачный мост без своей Wi-Fi инфры (дача, гараж 500м), мало нод (<19), трафик телеметрия (<10KB/с), и ты готов держать GND-шлюз.
- **Нет, если** нужна готовая экосистема датчиков/актуаторов, OTA из коробки, 100+ нод, Zigbee-экосистема — бери ESPHome + Zigbee2MQTT + LoRa (SX1262) или Wi-Fi mesh (ESP-MESH). DroneBridge — это всё-таки “телеметрия для дронов”, а не “умный дом”.

Для твоей задачи “по быстрому придумать канал для домашней автоматики” DroneBridge — **отличный временный канал**, но **не платформа**. Используй его как есть (Transparent + шлюз), а параллельно смотри на ESPHome как на конечную.

---

## 7. Как повторить проверку

```bash
idf.py set-target esp32c6 && idf.py build
# тесты парсера (есть в репо)
cmake -S test -B build/test && ctest --test-dir build/test
# фронт
cd frontend && npm install && npm run build
```

Дальше могу накидать `mqtt_to_udp` шлюз и патч `db_esp32_control.c` с хуком для домашних команд — скажи, какой путь (MVP 0 vs Hybrid) берём.

