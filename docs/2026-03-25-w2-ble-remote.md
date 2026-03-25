# ТЗ: Простой BLE-пульт для Yesoul W2 Walking Pad

**Дата:** 2026-03-25
**Статус:** Draft

## Цель

Минимальное приложение для управления дорожкой Yesoul W2 по Bluetooth Low Energy.
Одна страница, минимум кнопок: подключить, старт, стоп, скорость +/−.

Штатный пульт (маленький Bluetooth-брелок) сломался. Приложения Yesoul Fitness и PitPat слишком громоздкие — там нет простой кнопки "включить дорожку".

## Целевые платформы

**Вариант A (приоритетный) — Web Bluetooth (Chrome на MacBook M4 Pro):**
- Chrome поддерживает Web Bluetooth API на macOS
- Одна HTML-страница, открывается локально или через localhost
- Не требует установки, не требует зависимостей

**Вариант B (альтернативный) — Python + bleak (MacBook или Pixel 9 через Termux):**
- Библиотека `bleak` — кроссплатформенный BLE-клиент
- CLI или простой TUI на `textual`/`curses`
- `pip install bleak`

**Рекомендация:** начать с Варианта A (Web Bluetooth). Если окажется, что дорожка использует проприетарный протокол и нужен sniffing — переключиться на Вариант B.

## Архитектура BLE-соединения

### Фаза 1: Discovery (обнаружение сервисов)

Перед реализацией управления нужно определить, какой протокол использует дорожка.

**Шаг 1.** Сканировать BLE-устройства, фильтр по имени содержащему "Yesoul" или "YS".

**Шаг 2.** Подключиться и получить список GATT-сервисов и характеристик.

**Шаг 3.** Определить протокол:

| Сценарий | UUID сервиса | Что это значит |
|---|---|---|
| **FTMS (стандарт)** | `0x1826` | Fitness Machine Service — открытый стандарт, всё задокументировано |
| **Проприетарный** | `0xFFF0`, `0xFFF1` или подобный | Кастомный протокол Yesoul, нужен реверс-инжиниринг |

**В UI:** показать список найденных сервисов и характеристик для диагностики.

### Фаза 2a: Управление через FTMS (если обнаружен сервис 0x1826)

FTMS — стандарт Bluetooth SIG. Спецификация: https://www.bluetooth.com/specifications/specs/fitness-machine-service-1-0/

#### Ключевые UUID характеристик FTMS:

| Характеристика | UUID | Тип | Назначение |
|---|---|---|---|
| Fitness Machine Feature | `0x2ACC` | Read | Битовое поле поддерживаемых функций |
| Treadmill Data | `0x2ACD` | Notify | Текущая скорость, дистанция, время |
| Fitness Machine Control Point | `0x2AD9` | Write + Indicate | Отправка команд управления |
| Supported Speed Range | `0x2AD4` | Read | Мин/макс скорость, шаг |
| Machine Status | `0x2ADA` | Notify | Статус машины (старт, стоп, пауза) |

#### Протокол управления через Control Point (0x2AD9):

**Важно:** Перед отправкой команд нужно выполнить handshake:

1. **Request Control** — запрос управления (опкод `0x00`):
   ```
   Отправить: [0x00]
   Ожидать Response: [0x80, 0x00, 0x01] — success
   ```

2. **Start/Resume** (опкод `0x07`):
   ```
   Отправить: [0x07]
   ```

3. **Stop/Pause** (опкод `0x08`, с параметром):
   ```
   Stop:  [0x08, 0x01]
   Pause: [0x08, 0x02]
   ```

4. **Set Target Speed** (опкод `0x02`):
   ```
   Отправить: [0x02, low_byte, high_byte]
   ```
   Скорость в сотых км/ч, little-endian uint16.
   Пример: 3.0 км/ч = 300 = 0x012C → отправить `[0x02, 0x2C, 0x01]`

5. **Reset** (опкод `0x01`):
   ```
   Отправить: [0x01]
   ```

#### Чтение Supported Speed Range (0x2AD4):

Формат: 6 байт, little-endian uint16 × 3:
- Байты 0-1: минимальная скорость (в 0.01 км/ч)
- Байты 2-3: максимальная скорость (в 0.01 км/ч)
- Байты 4-5: шаг скорости (в 0.01 км/ч)

Пример: `64 00 58 02 0A 00` → min=1.0 км/ч, max=6.0 км/ч, step=0.1 км/ч

#### Чтение Treadmill Data (0x2ACD) через Notify:

Первые 2 байта — flags (битовое поле, определяет какие данные присутствуют).
- Если bit 0 = 0: Instantaneous Speed присутствует (uint16, в 0.01 км/ч)
- Далее опционально: дистанция, время, и т.д. — зависит от flags

### Фаза 2b: Управление через проприетарный протокол (если FTMS не найден)

Если дорожка отдаёт только кастомный сервис (типа `0xFFF0`):

1. В приложении сделать **режим отладки**: показывать все характеристики, подписываться на notify, отображать сырые байты в hex.

2. Для реверс-инжиниринга:
   - На Pixel 9 включить Bluetooth HCI Snoop Log (Developer Options → Enable Bluetooth HCI snoop log)
   - Подключить штатное приложение Yesoul Fitness, запустить дорожку
   - Выгрузить лог: `adb pull /sdcard/btsnoop_hci.log`
   - Открыть в Wireshark, фильтр по имени устройства

3. Как вариант, протокол может быть аналогичен WalkingPad (KingSmith):
   - Проект-референс: https://github.com/ph4r05/ph4-walkingpad
   - Используют сервис `0xFE00`, характеристики `0xFE01` (write) и `0xFE02` (notify)

## UI — Интерфейс

### Экраны

**Экран 1: Подключение**
- Кнопка "Подключить дорожку" → вызывает BLE scan
- Статус подключения (отключена / подключаюсь / подключена)
- Если подключено — показать имя устройства, обнаруженный протокол (FTMS / проприетарный / неизвестно)

**Экран 2: Управление (после подключения)**
- Текущая скорость (большие цифры, крупный шрифт)
- Кнопка **Старт**
- Кнопка **Стоп**
- Кнопка **+** (скорость вверх на 0.5 км/ч)
- Кнопка **-** (скорость вниз на 0.5 км/ч)
- Опционально: время, дистанция (если дорожка отдаёт через notify)

**Экран 3: Диагностика (скрытый, toggle по кнопке)**
- Список всех GATT-сервисов и характеристик (UUID, properties)
- Лог отправленных/полученных байтов в hex
- Сырые данные notify

### Требования к UI:
- **Крупные кнопки** — будет использоваться на ходу, с тредмила
- Адаптивно под мобильный экран (может открываться и на телефоне)
- Тёмная тема (чтобы не светить экраном)
- Минимум элементов, никаких менюшек

## Технические детали для Web Bluetooth (Вариант A)

### Scan и подключение:

```javascript
// Запрос устройства с фильтром по FTMS
const device = await navigator.bluetooth.requestDevice({
  filters: [
    { services: [0x1826] },                    // FTMS
    { namePrefix: 'Yesoul' },                  // По имени
    { namePrefix: 'YS' },                      // Альтернативное имя
  ],
  optionalServices: [0x1826, 0xFFF0, 0xFE00]   // На случай проприетарного
});

const server = await device.gatt.connect();
```

**Важно:** `navigator.bluetooth.requestDevice()` требует user gesture (клик).

### Подписка на Treadmill Data:

```javascript
const ftmsService = await server.getPrimaryService(0x1826);
const treadmillData = await ftmsService.getCharacteristic(0x2ACD);
await treadmillData.startNotifications();
treadmillData.addEventListener('characteristicvaluechanged', (event) => {
  const value = event.target.value; // DataView
  const flags = value.getUint16(0, true);
  // Если bit0 = 0 → speed в байтах 2-3
  const speed = value.getUint16(2, true) / 100; // км/ч
});
```

### Отправка команд через Control Point:

```javascript
const controlPoint = await ftmsService.getCharacteristic(0x2AD9);

// Обязательно подписаться на indicate перед записью
await controlPoint.startNotifications();
controlPoint.addEventListener('characteristicvaluechanged', handleResponse);

// Request Control
await controlPoint.writeValue(new Uint8Array([0x00]));

// Start
await controlPoint.writeValue(new Uint8Array([0x07]));

// Set speed 3.0 km/h = 300 = 0x012C
const speed = 300;
const buf = new Uint8Array([0x02, speed & 0xFF, (speed >> 8) & 0xFF]);
await controlPoint.writeValue(buf);

// Stop
await controlPoint.writeValue(new Uint8Array([0x08, 0x01]));
```

### Ограничения Web Bluetooth:
- Работает только в Chrome (и Edge, Opera). Не работает в Firefox и Safari.
- Требует HTTPS или localhost (или file:// не работает — нужен localhost)
- Каждый `requestDevice()` требует клика пользователя
- На Android Chrome Web Bluetooth тоже работает — можно открыть и на Pixel 9

### Запуск через localhost:

```bash
# Простейший вариант
python3 -m http.server 8080
# Открыть: http://localhost:8080/index.html
```

## Fallback: что делать если FTMS не работает

Если при подключении через nRF Connect или через Web Bluetooth discovery видно, что сервис `0x1826` отсутствует:

1. **Проверить PitPat** — это альтернативное приложение, которое Yesoul рекомендует. Если PitPat управляет дорожкой, значит протокол может быть совместим с WalkingPad/KingSmith.

2. **Снять BLE-дамп** (описано выше в Фазе 2b).

3. Разобрать протокол по hex-логу в диагностической панели.

## Референсы

- **FTMS спецификация:** https://www.bluetooth.com/specifications/specs/fitness-machine-service-1-0/
- **Web Bluetooth + FTMS treadmill (Decathlon):** https://github.com/Decathlon/domyos-developers/tree/main/ftms-treadmill-web-console
- **Web Bluetooth + FTMS контроллер:** https://github.com/janposselt/treadmill-monitor
- **Python BLE WalkingPad контроллер (на случай проприетарного протокола):** https://github.com/ph4r05/ph4-walkingpad
- **Python FTMS клиент (bleak):** https://nv1t.github.io/blog/treadmill-telemetry/
- **Гайд по проверке FTMS через nRF Connect:** https://kinni.co/does-my-treadmill-have-ftms-bluetooth/
- **BLE treadmill reverse engineering (Wireshark):** https://yvesdebeer.github.io/Treadmill-Bluetooth/
