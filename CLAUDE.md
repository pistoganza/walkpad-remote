# W2 — BLE-пульт для Yesoul W2 Walking Pad

## North Star

Простейшее приложение для управления дорожкой Yesoul W2 по Bluetooth — заменяет сломанный физический пульт. Открыл страницу → подключился → нажал «Старт». Без установки, без регистрации, без лишнего UI.

## Стек

- Web Bluetooth API, одна HTML-страница (`index.html`), Chrome на macOS/Android
- Без фреймворков, без сборщиков, vanilla JS + HTML + CSS
- Python http.server для localhost (с no-cache заголовками)

## Целевые устройства

- MacBook Pro M4 Pro (Chrome) — основной
- Google Pixel 9 (Chrome for Android) — вторичный

## Спецификация

- [docs/2026-03-25-w2-ble-remote.md](docs/2026-03-25-w2-ble-remote.md) — полная спека проекта

## Запуск

Открыть **[https://pistoganza.github.io/walkpad-remote/](https://pistoganza.github.io/walkpad-remote/)** в Chrome.

Деплой автоматический через GitHub Pages (push в main → GitHub Actions).

## BLE-протокол Yesoul W2

- Использует **FTMS** (Fitness Machine Service, сервис `0x1826`)
- Управление через Control Point (`0x2AD9`): Request Control → Start → Set Speed → Stop

### Нестандартное разрешение скорости (ВАЖНО!)

Yesoul W2 использует **нестандартный коэффициент скорости**:
- FTMS спецификация: 1 BLE-единица = 0.01 км/ч
- Yesoul W2 реально: 1 BLE-единица = **0.006 км/ч** (коэффициент 0.6)

Определено эмпирически:

| Отправлено (BLE) | Экран без коррекции | Дорожка показывает |
|---|---|---|
| 50 | 0.5 км/ч | 0.3 км/ч |
| 100 | 1.0 км/ч | 0.6 км/ч |
| 150 | 1.5 км/ч | 0.9 км/ч |

Конвертация в коде:
```javascript
const SPEED_RES = 0.006;
function bleToKmh(bleVal) { return bleVal * SPEED_RES; }
function kmhToBle(kmh) { return Math.round(kmh / SPEED_RES); }
```

### Прочие особенности

- Дорожка стартует на скорости **0.3 км/ч**
- Диапазон: 0.3 — 6.0 км/ч
- После Start нужна пауза ~500мс перед Set Speed
- BLE позволяет только одно соединение — закрыть Yesoul Fitness / PitPat перед подключением
- Auto-reconnect: Chrome запоминает разрешение на устройство, при перезагрузке пробует подключиться автоматически

## Функциональность (v9)

- Подключение к дорожке по BLE (с auto-reconnect)
- Start / Stop
- Пресеты скорости: 1.0 / 1.5 / 1.8 / 2.5 / 3.0 км/ч (автостарт)
- Точная подстройка +/- по 0.1 км/ч (с debounce 300мс)
- Отображение реальной скорости от дорожки (notify)
- Время и дистанция
- История тренировок (localStorage): дата, длительность, дистанция, средняя скорость
- Диагностическая панель (скрытая): GATT-сервисы, hex-лог BLE-пакетов
- Тёмная тема, крупные кнопки, адаптив под мобилку
- Инструкция подключения (встроена в страницу)
