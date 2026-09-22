# Лекция: Основы создания сервисов под Linux

**Дисциплина:** Основы программирования на языке Си  
**Тема:** `systemd`, unit-файлы, `systemctl`  
**Тип занятия:** лекция (2 ч)

---

## 1. Цели и задачи

**Цель:** научиться создавать собственные сервисы (демоны) в Linux и управлять ими через `systemctl`.

**Задачи:**
- Понять, что такое `systemd` и зачем он нужен.
- Освоить структуру unit-файла.
- Освоить команды `systemctl`.
- Научиться писать и регистрировать свой сервис.

**Компетенции:** ОК 01, ОК 02, ПК 1.1, ПК 1.2.

---

## 2. Что такое `systemd`

**`systemd`** — это система инициализации и менеджер сервисов в Linux. Она запускается первой (как **PID 1**) и управляет всеми остальными процессами.

**Что делает `systemd`:**

- Запускает сервисы при старте системы.
- Следит за их состоянием.
- Перезапускает упавшие.
- Ведёт логи (journal).
- Управляет монтированием, таймерами, сокетами.

**Ключевые понятия:**

| Термин | Что означает |
|--------|--------------|
| **Unit** | Единица управления (сервис, таймер, сокет) |
| **Unit-файл** | Конфигурация для unit |
| **Target** | Группа unit — состояние системы |
| **Journal** | Журнал логов от systemd |

**Типы unit-файлов:**

| Расширение | Что описывает |
|------------|---------------|
| `.service` | Сервис (программа) |
| `.timer` | Таймер (аналог cron) |
| `.socket` | Сокет |
| `.target` | Группа (состояние системы) |
| `.mount` | Точка монтирования |

---

## 3. Структура unit-файла

Unit-файл состоит из **трёх основных секций**:

```ini
[Unit]
Description=Мой сервис
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/app/main.py
Restart=always

[Install]
WantedBy=multi-user.target
```

### 3.1. Секция `[Unit]`

Общая информация и зависимости:

| Директива | Что означает |
|-----------|--------------|
| `Description` | Описание сервиса |
| `After` | Запустить **после** указанных unit |
| `Before` | Запустить **до** указанных unit |
| `Requires` | **Жёсткая** зависимость — без неё не запускать |
| `Wants` | **Мягкая** зависимость — если не запустится, не критично |

### 3.2. Секция `[Service]`

Как запускать сервис:

| Директива | Что означает |
|-----------|--------------|
| `Type` | Тип сервиса (`simple`, `forking`, `oneshot`, `notify`) |
| `ExecStart` | Команда запуска (**абсолютный путь!**) |
| `ExecStop` | Команда остановки |
| `ExecReload` | Команда перезагрузки конфигурации |
| `Restart` | Когда перезапускать: `no`, `always`, `on-failure` |
| `RestartSec` | Сколько секунд ждать перед перезапуском |
| `User` / `Group` | От какого пользователя запускать |
| `WorkingDirectory` | Рабочий каталог |
| `Environment` | Переменные окружения |

**Значения `Type`:**

| Type | Что означает |
|------|--------------|
| `simple` | Основной процесс — то, что запущено (по умолчанию) |
| `forking` | Классический демон: родитель запускает ребёнка и завершается |
| `oneshot` | Одноразовая команда, выполняется и завершается |
| `notify` | Сервис сам сообщает о готовности через API |

### 3.3. Секция `[Install]`

Как включать сервис:

| Директива | Что означает |
|-----------|--------------|
| `WantedBy` | К какому target привязать (обычно `multi-user.target`) |
| `Alias` | Альтернативное имя |

---

## 4. Полный пример

Пусть есть Python-скрипт `/opt/app/main.py`, который нужно сделать сервисом.

**Шаг 1. Создать unit-файл:**

```bash
sudo nano /etc/systemd/system/myapp.service
```

**Шаг 2. Содержимое:**

```ini
[Unit]
Description=My Python Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/app/main.py
Restart=always
RestartSec=5
User=www-data
WorkingDirectory=/opt/app

[Install]
WantedBy=multi-user.target
```

**Шаг 3. Перезагрузить конфигурацию systemd:**

```bash
sudo systemctl daemon-reload
```

**Шаг 4. Запустить сервис:**

```bash
sudo systemctl start myapp
```

**Шаг 5. Проверить статус:**

```bash
sudo systemctl status myapp
```

**Шаг 6. Включить автозапуск:**

```bash
sudo systemctl enable myapp
```

---

## 5. Команды `systemctl`

| Команда | Что делает |
|---------|-----------|
| `systemctl start имя` | Запустить |
| `systemctl stop имя` | Остановить |
| `systemctl restart имя` | Перезапустить |
| `systemctl reload имя` | Перечитать конфигурацию (без перезапуска) |
| `systemctl status имя` | Показать статус |
| `systemctl enable имя` | Включить автозапуск |
| `systemctl disable имя` | Отключить автозапуск |
| `systemctl is-enabled имя` | Проверить автозапуск |
| `systemctl list-units --type=service` | Список сервисов |
| `systemctl cat имя` | Показать unit-файл |
| `systemctl edit имя` | Изменить через drop-in |

---

## 6. Просмотр логов

Логи systemd собирает в **journal**. Смотреть так:

```bash
journalctl -u myapp.service            # все логи сервиса
journalctl -u myapp.service -f         # следить в реальном времени
journalctl -u myapp.service --since "10 min ago"
journalctl -u myapp.service --until "1 hour ago"
```

⚠️ **Без `-u`** будет вывод всех логов системы. `-u` фильтрует по имени сервиса.

---

## 7. Drop-in — переопределение без правки оригинала

Если нужно изменить **готовый** unit-файл (например, из пакета), его не правят напрямую. Создают **drop-in**:

```bash
sudo systemctl edit myapp.service
```

Это создаст файл `/etc/systemd/system/myapp.service.d/override.conf`, где можно переопределить параметры:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/python3 /opt/app/new_main.py
Environment=DEBUG=1
```

⚠️ **Первая строка `ExecStart=` пустая** — она **сбрасывает** старое значение. Без неё добавится второй `ExecStart`, что вызовет ошибку.

---

## 8. Пользовательские сервисы

Сервисы можно создавать **без root** — только для своего пользователя.

**Каталог:**

```
~/.config/systemd/user/
```

**Создание:**

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/myservice.service
```

**Управление:**

```bash
systemctl --user start myservice
systemctl --user status myservice
systemctl --user enable myservice
```

⚠️ Пользовательские сервисы запускаются **при входе** пользователя и останавливаются **при выходе** (если не включить lingering).

---

## 9. Таймеры (`.timer`)

`.timer` — аналог `cron` для systemd.

**Пример: запуск каждые 2 часа по будням**

Файл `update.timer`:

```ini
[Unit]
Description=Run update every 2 hours

[Timer]
OnCalendar=Mon..Fri 00/2
Unit=update.service

[Install]
WantedBy=timers.target
```

Файл `update.service` — что делать:

```ini
[Unit]
Description=Update job

[Service]
Type=oneshot
ExecStart=/opt/app/update.sh
```

**Включение:**

```bash
sudo systemctl enable update.timer
sudo systemctl start update.timer
sudo systemctl list-timers
```

---

## 10. Частые ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Unit not found` | Файл не в `/etc/systemd/system/` | Переместить или создать там |
| `ExecStart` не найден | Относительный путь | Использовать абсолютный |
| Изменения не применяются | Забыли `daemon-reload` | `sudo systemctl daemon-reload` |
| Сервис падает | Смотрите `journalctl -u имя` | Изучить логи |
| `enable` не работает | Нет секции `[Install]` | Добавить `WantedBy` |
| Ошибка прав | `User` не существует | Проверить пользователя |
| Drop-in не действует | Забыли `ExecStart=` (пустой) | Добавить первую строку |

---

## 11. Итоги

- **`systemd`** — PID 1, менеджер сервисов.
- **Unit-файл** состоит из `[Unit]`, `[Service]`, `[Install]`.
- **`ExecStart`** — абсолютный путь к программе.
- **`systemctl`** — start, stop, enable, status.
- **`journalctl`** — просмотр логов.
- **Drop-in** — переопределение без правки оригинала.
- **Пользовательские сервисы** — без root.
- **Таймеры** — замена `cron`.

---

## 12. Вопросы для самоконтроля

1. Что такое `systemd` и зачем он нужен?
2. Из каких секций состоит unit-файл?
3. Чем `Type=simple` отличается от `Type=forking`?
4. Почему `ExecStart` требует абсолютный путь?
5. Что делает `systemctl enable`?
6. Зачем нужен `daemon-reload`?
7. Как посмотреть логи сервиса?
8. Что такое drop-in?
9. Чем пользовательский сервис отличается от системного?
10. Как создать таймер в systemd?

---

## 13. Домашнее задание

1. Написать простой скрипт, выводящий текущее время.
2. Создать unit-файл для этого скрипта.
3. Запустить, посмотреть статус.
4. Включить автозапуск.
5. Посмотреть логи через `journalctl -u`.
6. Создать таймер, запускающий скрипт каждую минуту.
7. Ответить на вопросы самоконтроля.

---

## 14. Литература

1. Лав Р. «Linux. Системное программирование». Глава 8.
2. `man 5 systemd.service`, `man 5 systemd.unit`.
3. `man 1 systemctl`, `man 1 journalctl`.
4. Официальный сайт: systemd.io.

---

