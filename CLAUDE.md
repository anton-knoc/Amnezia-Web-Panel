# CLAUDE.md — Amnezia Web Panel (fork пользователя)

## Что это

Веб-панель (Flask) для управления AmneziaWG / WireGuard / Xray / Telemt / AmneziaDNS / AdGuard Home / SOCKS5 на удалённых Ubuntu-серверах через один дашборд. Апстрим: https://github.com/PRVTPRO/Amnezia-Web-Panel

## Репозиторий

- **origin** = `PRVTPRO/Amnezia-Web-Panel` (апстрим, официальный)
- **fork** = `anton-knoc/Amnezia-Web-Panel` (форк пользователя, anton_knoc / anton_k)
- Рабочая ветка: `fix/awg2-dynamic-subnet-start-script` (HEAD `093c140`)
- Локальный `main` устарел (на базе v1.4.4, `a62f958`)
- `origin/dev` — дев-ветка апстрима

## Судьба кастомных изменений — ПОЛНОСТЬЮ СЛИТЫ В АПСТРИМ

Merge-base локальной ветки и `origin/main` = сам HEAD (`093c140`), т.е. ветка —
прямой предок апстрима. Все кастомные изменения вошли через **PR #70**
(merge `240dbb6`) и доработаны апстримом:

- Динамический SUBNET в start.sh (`093c140`) — сохранён в `managers/awg_manager.py`,
  расширен до dual-stack IPv6 (PR #86) и перенесён в `_render_start_script()` (PR #123)
- Перегенерация start.sh при сохранении конфига — сохранена (save_server_config)
- IPAM: заполнение дырок в подсети (PR #80), резервирование IP отключённых клиентов (PR #728dd5b)
- Туннели (Local Server/Cloudflare/ngrok, `a62f958`) — в апстриме, 113 упоминаний в app.py

Ветка `fix/awg2-dynamic-subnet-start-script` **более не нужна** — после синхронизации
форка с origin/main её можно удалить.

## Незакоммиченная работа (актуальная, апстримом не покрыта)

6 файлов, +50/−12: опциональное поле **Subnet Address** при установке протокола
(дефолты 10.8.1.0/24 AWG, 10.8.2.0/24 WG):

- `app.py` — `subnet_address` в `InstallProtocolRequest` и обработчике install
- `managers/awg_manager.py`, `managers/wireguard_manager.py` — прокидывание в
  `install_protocol` / `_configure_container`
- `templates/server.html` — поле в инсталл-модалке
- `translations/en.json`, `ru.json` — 2 аддитивных ключа

## Версии

| Что | Версия | Комментарий |
|---|---|---|
| Сервер 38.180.212.31 | **v1.4.4** | systemd-сервис `amnezia-panel.service`, порт 5000, каталог `/opt/Amnezia-Web-Panel` (НЕ git-clone; app.py байт-идентичен релизу v1.4.4), развёрнут 2026-07-03 |
| GitHub (последний релиз) | **v1.6.6** | 2026-09-09 (обратите внимание: константа `CURRENT_VERSION` в app.py релиза отстаёт — в v1.6.6 указано v1.6.5) |
| origin/main | v1.6.5 (dbccc1e) | ~290 коммитов поверх локальной ветки (96 файлов, +25к/−3к) |
| Локальная ветка | v1.4.4 + 1 коммит | задеплоенный на сервер код на локальную ветку НЕ похож — сервер чистый v1.4.4 без subnet-фикса |

## Прод-сервер (осторожно!)

- `ssh 38.180.212.31` — креды подставляются из `~/.ssh/config` (не добавлять
  порт/пользователя/ключ вручную)
- **Боевая инсталляция с живыми клиентами.** Только read-only команды по умолчанию;
  рестарты/изменения — только по явному запросу пользователя.
- Панель: systemd `amnezia-panel.service`, venv в `/opt/Amnezia-Web-Panel/venv`, порт 5000
- AWG-контейнер: `amnezia-awg2` (Docker), аптайм ~2 месяца

## Главные изменения апстрима v1.4.4 → v1.6.x (290 коммитов)

- **AWG**: 3.1, dual-stack IPv6, DKMS-модуль, host tuning (BBR, nf_conntrack, ulimit),
  per-peer лимиты скорости (tc), MTU/DNS/I1–I5 junk-пакеты в API+UI
- **Новое**: exit-ноды (`exit_manager.py`, `exit_link_service.py`), импорт из
  wg-easy/amnezia-wg-easy + мульти-инстансы, Telegram self-service, переименование
  peer'ов, восстановление бэкапов, роли юзеров
- **Инфра**: пул/хардненинг SSH-соединений, PWA (`pwa.py`), design system +
  мобильный layout, `tests/` (~25 тестов), GHCR-публикация, `DATA_FILE` env,
  рефакторинг генерации Dockerfile/start.sh (25f9d2c)

## AWG 3.1 (awg3) в апстриме — миграция клиентов

- В панели `awg3` — **отдельный протокол** (`BASE_PROTOCOLS = ['awg', 'awg2', 'awg3', 'awg_legacy', ...]`),
  ставится как отдельный контейнер `amnezia-awg3`, поддерживает мульти-инстансы.
  Это НЕ обновление существующих awg/awg2.
- **Клиенты не переезжают автоматически**: у AWG 3.1 новый набор ключей
  (`HeaderProtectionKey`, `RandomTrailers`, `DisableCookies`, `ContentPaddingAddition`, ...),
  старые клиенты AWG 2.x с ним несовместимы. Миграция = установить `awg3`
  (новый контейнер/порт) → сгенерировать **новые конфиги** → раздать клиентам.
  Нужна версия клиентского приложения с поддержкой 3.1.
- Существующие awg2-клиенты продолжают работать без изменений.
- Нюанс сервера: нужен kernel-модуль amneziawg 3.0+; на старом модуле панель
  автоматически форсирует userspace (`amneziawg-go`, WG_FORCE_USERSPACE, issue #113).
  При заданном HeaderProtectionKey junk-размеры S1–S4 должны быть ≥ 12.

## План обновления сервера v1.4.4 → latest

1. **Закоммитить незакоммиченную фичу** (Subnet Address) на текущей ветке
2. Обновить `main`: fast-forward до `origin/main`, замержить в fork
3. **Перебазировать коммит с фичей** на origin/main. Ожидаемые конфликты:
   - `app.py` — **высокий** (обработчик install переписан, файл вырос с ~2100 до ~3450 строк)
   - `templates/server.html` — **высокий** (инсталл-модалка полностью переделана)
   - `managers/awg_manager.py` — средне-высокий (сигнатуры install_protocol/_configure_container
     расширены: mtu/dns/special_junk/ipv6 — фича концептуально ложится рядом)
   - `managers/wireguard_manager.py` — средний
   - translations — низкий (аддитивные ключи)
4. Прогнать тесты (`tests/`), проверить установку протокола с полем Subnet Address
5. На сервере: **резервная копия** `/opt/Amnezia-Web-Panel` (и файла данных панели) →
   остановить `amnezia-panel.service` → заменить код → запустить → проверить
   панель и живых клиентов. AWG-контейнер `amnezia-awg2` не трогать.
6. Подтянуть изменения start.sh-логики на сервер в соответствии с новым апстримом
   (генерация скрипта теперь в `_render_start_script()`)
