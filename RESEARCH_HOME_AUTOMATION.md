# DroneBridge-for-ESP32 — исследование применимости для домашней автоматики

**Дата:** 2026-10-19

**База:** `DroneBridge-for-ESP32-Arduino-mode` v2.4.1 (ESP-IDF 5.4.4, chip ESP32/C3/C6/S2/S3), ветка `master` @ `ed1b9ef`

**Цель:** ответить на заметки — можно ли быстро взять DroneBridge как радиоканал для датчиков: опрос → MQTT, загрузка профилей/скриптов, MQTT → broadcast, своё ядро + обновление по воздуху.

> TL;DR для спешащих — **DroneBridge как готовый “радиомост” очень хорош для 1–2 недель MVP**, но **как полноценная платформа домашней автоматики — потребует форка и переработки**. Быстрее всего завести канал: ESP-NOW AIR/GND в `Transparent` и MQTT-шлюз на ESP32-GND как часть Hybrid Core либо внешний шлюз через UART GND; TCP 5760/UDP 14550 существуют в режимах Wi-Fi AP/STA, **не в ESP-NOW**. Нативный MQTT-клиент внутри ESP-NOW-GND требует переделки Wi-Fi init под одноканальное STA+ESP-NOW; файловую систему для скриптов и OTA придётся проектировать отдельно. В open-source ветке OTA сознательно отсутствует — OTA есть только в платной DLSE; для open-source придётся ввести OTA-раздел самостоятельно.

---

## 1. Что такое DroneBridge сегодня (факты)

Малая прошивка-мост: `UART/JTAG ↔ Radio`. На ESP32 поднимает либо Wi-Fi AP/STA/LR, либо ESP-NOW LR (connectionless broadcast до ~1 км [6](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/drone-swarm-control)), либо BLE. Данные гоняет прозрачно или с лёгким парсингом MAVLink/MSP/LTM и инжектом `RADIO_STATUS` для GCS.

**Архитектура (упрощённо):**

```
ESP-NOW: [датчик/FC] --UART--> [ESP32 AIR] --ESP-NOW broadcast--> [ESP32 GND] --UART--> [шлюз MQTT]
Wi-Fi:   [датчик/FC] --UART--> [ESP32 STA/AP] --TCP 5760 / UDP 14550--> [шлюз MQTT/GCS]
HTTP /api/* и mDNS доступны только в некоторых Wi-Fi/BLE режимах, не в ESP-NOW; параметры — NVS.
```

- **Режимы** `SYS_ESP32_MODE`: 1 AP, 2 STA, 3 AP-LR, 4 ESP-NOW AIR, 5 ESP-NOW GND, 6 BLE [2](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/configuration)
- **Сериализация:** `SERIAL_TEL_PROTO`: 1 MSP/LTM, 4 MAVLink, 5 Transparent (любой байтовый поток)
- **Сеть в Wi-Fi режимах:** TCP 5760, UDP 14550; служебный multicast 232.10.11.12:1606 используется в AP-LR/STA для RSSI; HTTP REST включён не во всех режимах.
- **Шифрование:** WPA2 PSK в Wi-Fi, AES-GCM 256 в ESP-NOW (соль `GenSalt147894562`, PBKDF2-SHA256 10k итераций — см. `db_esp_now.c`).
- **Frontend:** `frontend/` собирается в один `index.html` с inline CSS/JS (специально, чтобы веб-сервер не грузил пачку файлов).
- **Партиции:** `partitions.csv` → `factory @ 0x10000 1536K` + `spiffs www size 0x30000`. **OTA-партиций нет**. Для OTA придётся менять таблицу.

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

- **MVP (1 неделя):** оставь DroneBridge untouched, датчик шлёт в UART строку `{"id":"kitchen","t":22.4,"h":58}\n`. В режиме ESP-NOW GND отдаёт данные через UART; внешний шлюз слушает UART, а не TCP/UDP. Если вместо этого используется обычный Wi-Fi AP/STA, шлюз может слушать TCP 5760/UDP 14550. ESP-NOW GND не работает одновременно как STA/AP с IP-сокетами в текущем коде. Так делают Swarm-гайды DroneBridge — ESP-NOW ноды стекаются к одной GND-шлюзу [6](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32/drone-swarm-control).
- **Следующий шаг — B:** добавь свой сенсор-драйвер как отдельный компонент `components/sensor_hub/`. В `app_main()` после `db_start_control_module()` запускай `xTaskCreate(sensor_task, ...)`. Сенсор-таск с интервалом 1–5с читает BME280/DS18B20 и форматирует. Для передачи из своей задачи используй `db_send_to_all_radio_clients()` (дождись инициализации ESP-NOW очередей); она режет на `DB_ESPNOW_PAYLOAD_MAXSIZE` (215 байт), **без протокола сборки и подтверждения**.

> Важно: `DB_SERIAL_PACK_SIZE_MAX = 1023`, `SERIAL_PACK_SIZE` по умолчанию 576, `serial_timeout` 20мс. Для сенсорного JSON ставь `pack_size=128–256` и `proto=Transparent` — иначе MAVLink-парсер будет ждать валидный фрейм и дропать.

**Вывод в MQTT — два места:**

1. **Внешний шлюз за UART ESP-NOW GND (рекомендовано для MVP):** один MQTT-клиент на всех. Для однокристального GND+MQTT нужен отдельный дизайн Wi-Fi/ESP-NOW coexistence (ESP-NOW и STA должны работать на одном канале), которого эта прошивка не предоставляет.
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

- 27 параметров в NVS (`db_parameters.h: DB_PARAM_TOTAL_NUM 27`), доступны через `GET/POST /api/settings` (JSON, только режимы с HTTP) и через MAVLink `PARAM_*` (только 18 числовых, строковые — только REST).
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

Текущий путь ESP-NOW: `UART GND → db_send_to_all_radio_clients() → зашифрованный ESP-NOW broadcast → очередь на AIR → UART AIR`. В обратном направлении AIR → GND аналогично. MQTT-клиента, IP-сокета и HTTP-сервера в режиме ESP-NOW **нет** (`main/main.c`, ветки старта режимов). Поэтому отправить UDP с Raspberry Pi в IP ESP-NOW-GND невозможно. Wi-Fi AP/STA и ESP-NOW AIR/GND — альтернативные режимы, не последовательные звенья одной сети.

Для **Hybrid Core**: на GND в `main/` подключить MQTT к брокеру потребует спроектировать работу STA+ESP-NOW на одном канале и жизненный цикл Wi-Fi; простой `mqtt_task` без изменения `db_init_wifi_espnow()` не заработает. Более простой путь: мост MQTT↔UART на внешнем хосте, а в прошивке AIR добавить перехват команд в `control_module_esp_now()` перед `write_to_serial()`; маршрутизировать по application-level `node_id`, сделать `req_id`, ACK, дедупликацию и таймаут. Никакого прямого unicast-переключения в коде нет.

`db_send_to_all_radio_clients()` делит длинные сообщения на фрагменты до 215 байт, **но приёмник не собирает их обратно**. Отдельно код GND ведёт таблицу **19 записей метрик** по MAC (RSSI/потери) — это предел одного служебного пакета, **не лимит ESP-NOW-подключений**. Данные от последующих узлов всё равно принимаются и передаются в UART (если хватает эфира/очереди), но качество для них не отслеживается. См. `main/db_esp_now.c:update_peer_list`, `db_espnow_process_rcv_data`.

Текущая галка AIR→AIR — только для MAVLink, не mesh-маршрутизатор и не механизм ретрансляции Transparent-команд.

### 2.4 Скрещивание DroneBridge движка + своё ядро

**Где живёт логика DroneBridge:**

- `app_main()` → `db_init_wifi_*` / `db_start_espnow_module()` → `db_start_control_module()` → два FreeRTOS-пула: Control и ESP-NOW.
- Control крутит `while(1)` с `read_process_serial_link()` + приём TCP/UDP/BLE и очередь `db_uart_write_queue`.
- Все параметры — глобали `DB_PARAM_*`, доступ через `extern`.

**Стратегии скрещивания:**

**Вариант 1 — “Тонкий мост” (рекомендован для старта):** не трогать DroneBridge, свой код — отдельный процесс на GND-шлюзе (малина/второй ESP). DroneBridge остаётся чистым радиомостом. Плюс: 0 риска сломать радио, обновления DroneBridge накатываешь штатно. Минус: +1 железка.

**Вариант 2 — “Hybrid Core” (оптимальный среднесрок):** форк `main/` + свой компонент `components/homeauto_core/`. Свой core — FreeRTOS-таски `sensor_task`, `actuator_task`, `mqtt_task` с очередями. Общение с DroneBridge через уже существующие `db_send_to_all_radio_clients()` и `db_uart_write_queue`. Важно не трогать Wi-Fi init (там `esp_netif_create_default_wifi_ap`, `esp_wifi_init`), а добавить `sensor_task` после. Доступная RAM и нагрузка зависят от платы/сборки — измерить heap high-water mark и размеры стека на устройстве, не закладывать фиксированные цифры без замера. MQTT через STA в ESP-NOW режиме потребует доработки радиослоя и общей настройки канала.

**Вариант 3 — “Fork + рефактор”:** вынести DroneBridge в либу `libdronebridge.a` и писать свой `app_main`. Долго, но чисто для продукта. Требует понимания `idf.py set-target` и `Kconfig.projbuild`.

**Практические правила гибрида:**

- Работа в этом репозитории остаётся в текущей ветке `arena/f7b2024a-dronebridge-for-esp32-arduino`. Локальные `AGENTS.md` исключают добавление OTA/C5/IDF 5.5+; OTA здесь обсуждается только как отдельный архитектурный вариант, без реализации.
- Не меняй `CONFIG_LWIP_MAX_SOCKETS=16` и `max_uri_handlers=9` без нужды.
- Сенсор-таски — низкий приоритет (например 2), Control и ESP-NOW в текущем коде — приоритет 5; проверить на плате загрузку и потери.
- Для периодического опроса сенсоров достаточно `vTaskDelayUntil`; не блокируй Wi-Fi callback и существующие control/espnow tasks. Текущие MAVLink таймеры реализованы FreeRTOS software timers.

Схематичный пример врезки своей задачи (не готовая сборка; требуется инициализация датчиков/очередей и обработка ошибок):
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

**Для отдельного продукта с OTA** нужно проектировать корректную OTA-разметку под фактический объём flash и размер приложения (проверять CSV генератором IDF на пересечения/выравнивание), две app-ячейки, `otadata`, а также авторизованный механизм доставки, проверки подписи/хэша, rollback и восстановления питания. Нельзя переносить приведённую ранее черновую таблицу партиций: её смещения пересекались. Это **не реализовано** в открытой прошивке и не будет добавляться в данный репозиторий (см. `AGENTS.md`). Обновление AIR через ESP-NOW потребовало бы отдельного chunk/ACK/resume протокола; HTTP по IP в существующем ESP-NOW режиме отсутствует.

**Вывод:** для быстрого старта OTA не делай — используй флешер + профили. Для продукта заложи OTA-партиции сразу, но реализуй после MVP.

---

## 3. Проверяемые ограничения текущей реализации

- ESP-NOW LR = фиксированный радиоканал, без IP и без автоматического unicast/опроса.
- 19 — размер таблицы RSSI/packet-loss в одном служебном кадре, а не максимум AIR-узлов.
- Очереди по 6 элементов, нет прикладного ACK, повторной передачи и сборки разбитых сообщений.
- Предел 215 байт полезного содержимого в одном ESP-NOW кадре при `ESP_NOW_MAX_DATA_LEN=250`.
- Никаких подтверждённых измерений дальности/масштабирования именно этой сборки здесь не проводилось. Упоминания частных багов, нагрузок RAM и сроков в предыдущей версии отчёта были предположениями и удалены.

---

## 4. Рекомендация для Hybrid Core

1. Выбрать железо и канал: тестировать пару ESP32-C3/C6; одинаковый канал и групповой `espnow_secret` на GND и AIR.
2. В прошивке AIR добавить `sensor_task` и обработчик прикладных сообщений после проверки GCM, не забирать `db_uart_write_queue` вторым конкурентным consumer-ом.
3. В GND выбрать одно из двух: внешний MQTT↔UART хост (проще и надёжнее) **или** отдельный инженерный этап STA+ESP-NOW coexistence на одном канале (одна ESP32, но меняется Wi-Fi init). Не предполагать доступность UDP 14550/REST из ESP-NOW режима.
4. В payload использовать бинарный envelope `{version, type, node_id:uint32, req_id:uint32, length, body}`; всем широковещательный кадр, только адресат обрабатывает запрос; ответ адресован через `req_id` и `node_id` поверх такого же broadcast. ACK/таймаут/retry с разбросом задержек — собственный протокол, не штатная функция DroneBridge.
5. Провести измерение пропускной способности, RSSI, packet-loss, свободной RAM и устойчивости при увеличении числа узлов. Не ориентироваться на «19 узлов максимум» — ограничения определяет эфир и код обработки.

---

## 5. План развития (оценка)

| Этап | Что | Срок | Риск |
|---|---|---|---|
| **MVP 0** | 2× ESP32 + Transparent + внешний GND UART↔MQTT шлюз | Оценивать по прототипу | Низкий |
| **MVP 1** | Hybrid Core, сенсор-таск I²C, NVS-профили (REST только в Wi-Fi режиме) | Оценивать по прототипу | Средний |
| **MVP 2** | При необходимости STA+ESP-NOW GND, QoS приложения, GPIO | Отдельный этап | Высокий |
| **Продукт** | OTA только вне этого репозитория/по отдельному решению, надёжность и безопасность | Отдельный проект | Высокий |

---

## 6. Итог — стоит ли DroneBridge для дома?

- **Да, если** нужен дальнобойный прозрачный мост без своей Wi-Fi инфры (дача, гараж 500м), трафик, проверенный экспериментально на своём канале, и ты готов держать GND-шлюз.
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

Следующий шаг для Hybrid Core — прототип узла/шлюза и измерение радиоканала с адресацией и ACK на прикладном уровне.


---

## 8. Масштаб, опрос и OTA: DLSE ≠ ESP-NOW обычной прошивки

**Проверено 2026-10-19:** [DLSE Overview](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/overview-drone-light-show-edition) рисует **Skybrush Live → Ethernet → Wi-Fi AP → DLSE ESP32 (STA) → UART → flight controller**. [DLSE Configuration](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/configuration) предлагает лишь AP/STA, для шоу обычно STA. Это отдельная коммерческая прошивка, не код этого репозитория. Сайт [DLSE](https://drone-bridge.com/dlse/) говорит об эксплуатации флотов 1000+, но **не подтверждает 10 000 узлов на одном AP/частоте, ни опрос пачками по 19**. Wi-Fi 6/OFDMA на поддерживаемых устройствах и планирование AP/каналов — другая архитектура, чем ESP-NOW LR.

| Слой | Открытая прошивка: ESP-NOW AIR/GND | DLSE (по открытой документации/Support Suite) |
|---|---|---|
| Радио | ESP-NOW + LR, один настроенный канал, без ассоциации с AP | Wi-Fi STA каждого устройства ↔ инфраструктурные AP; AP режим для настройки |
| Адресация | `esp_now_send(FF:FF:FF:FF:FF:FF)`; `origin` GND/AIR, физический source MAC, **нет target node ID** | IP-адрес узла; MAVLink sysid/component id внутри телеметрии |
| Сбор телеметрии | AIR самостоятельно отдаёт UART-данные; GND принимает; нет опросчика | FC передаёт MAVLink → DLSE bridge → UDP/TCP через AP → GCS; политика расписания FC/Skybrush не установлена по открытым исходникам DLSE |
| Сервисная телеметрия | GND периодически вещает *один список* MAC/RSSI/loss до 19 записей; AIR выбирает себя по MAC | MAVLink `RADIO_STATUS`, прочие параметры/HTTP stats; точный закрытый radio runtime не аудитирован |
| OTA | отсутствует | отдельная Wi-Fi/IP-процедура: MAVLink broadcast для discovery → HTTP GET `/api/system/info` → HTTP POST binary `/update/www`, затем `/update/firmware` по IP каждого узла |

**Важно про 19:** `main/db_espnow_limits.h` ограничивает **массив метрик** для одного служебного пакета (тип `DB_ESP_NOW_PACKET_TYPE_INTERNAL_TELEMETRY`), не адресное пространство и не количество установленных «соединений». Приём DATA не отвергает узел при переполнении списка; сохраняются ограничения ёмкости эфира/очередей/памяти. Нет ни последовательного обхода по 19, ни переключения на отдельный радиоканал для ответа: в `main/main.c:465–485` канал фиксирован, в `main/db_esp_now.c:219` инициализируется единственный broadcast peer. ESP-NOW callback об отправке не равен прикладному ACK у адресата.

**Пример `#1042, канал 8`:** канал 8 в `esp_wifi_set_channel(8, ...)` — общая радиочастота всей группы, не выделенный сеанс. MAVLink `sysid` 8-битный: `1042` не может быть MAVLink SYS ID; для своего протокола введи `uint32_t node_id`, который проверяет каждый узел *после* общей аутентификации пакета. На том же канале узел 1042 ответит с `req_id`; остальные промолчат. Без временных окон/jitter при одновременных ответах получишь коллизии. Для большого флота нужна сегментация по AP/радиоканалам и телеметрийный бюджет, а не магическое масштабирование через broadcast.

**Проверяемые детали OTA DLSE:** открытый [`batch_ota_update_allinone.py`](https://github.com/DroneBridge/DLSECommercialSupportSuite/blob/main/batch_ota_update_allinone.py) делает discovery и **поочередный** upload на найденные IP; [`DroneBridgeCommercialSupportSuite.py`](https://github.com/DroneBridge/DLSECommercialSupportSuite/blob/main/DroneBridgeCommercialSupportSuite.py) отправляет по UDP broadcast MAVLink `HEARTBEAT` + `COMMAND_LONG(MAV_CMD_REQUEST_MESSAGE, AUTOPILOT_VERSION=148)` на порт 14555; слушает 14550, фильтрует `comp_id=240`. Затем двоичные `www.bin` и `db_esp32.bin` передаются `POST /update/www` и `POST /update/firmware` **обычным HTTP по IP**, это не MAVLink-бинарник и не ESP-NOW. [`OpenAPI`](https://github.com/DroneBridge/DLSECommercialSupportSuite/blob/main/api_definition/openapi_definition.yaml) описывает эти endpoints. Реализация приёма OTA на DLSE стороне закрыта, поэтому точный внутренний механизм проверки/rollback по этому проекту не доказан. [Safety guideline](https://dronebridge.gitbook.io/docs/dronebridge-for-esp32-drone-light-show-edition/safety-and-integration-guideline) запрещает обновление вооружённого/летящего дрона; отдельно отмечает ограничения проверки состояния FC при потере MAVLink.

**Оценка масштаба только как sanity check:** при 10 000 узлов и *одном* сообщении 50 байт в секунду это **4 Мбит/с только данных** (без MAC/преамбулы/коллизий). Для ESP-NOW LR порядка 250 кбит/с *gross* это физически не поместится на одном канале. Wi-Fi-инфраструктура с несколькими AP/каналами и расписанием может масштабироваться, но точные числа допустимых узлов и режимы Skybrush нужно измерять на реальной сети, не выводить из 19-элементной таблицы ESP-NOW.
