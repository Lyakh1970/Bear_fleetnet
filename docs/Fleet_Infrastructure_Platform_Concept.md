# Fleet Infrastructure Knowledge & Configuration Platform

## 1. Обзор проекта

**BEAR_FLEETNET** — это проект создания централизованной self-hosted платформы для учёта, мониторинга, хранения конфигураций и AI-assisted поиска по сетевой и IT-инфраструктуре флота.

Платформа предназначена для эксплуатации в реальных условиях рыболовного флота, где одновременно присутствуют:

- судовые Cisco/OPNsense сети;
- Starlink / Iridium / mixed WAN;
- MGMT VLAN;
- CCTV VLAN;
- судовые серверы и edge-hosts;
- навигационные и промысловые системы;
- NMEA/UDP/serial gateways;
- ограниченный onboard IT staff;
- необходимость удалённой поддержки с берега.

Главная цель проекта — уйти от разрозненных Google Sheets, локальных папок, отдельных Cursor-проектов и ручного поиска по конфигам к единой системе, где есть:

- понятный human-friendly inventory;
- история изменений конфигураций;
- мониторинг того, что реально сейчас работает;
- AI-интерфейс для быстрого поиска и анализа.

---

## 2. Общая архитектурная идея

Платформа строится вокруг центрального сервера **BearCore LP (BCL)**.

BCL рассматривается как центральная точка self-hosted сервисов:

```text
BearCore LP / BCL
 ├── Baserow
 ├── PostgreSQL
 ├── Gitea
 ├── Oxidized
 ├── LibreNMS
 ├── AI/RAG layer, future phase
 └── deployment scripts / automation
```

Связь с судами должна выполняться только через защищённый overlay/VPN layer:

```text
BCL
 ↓
Tailscale / ZeroTier / WireGuard
 ↓
Onboard edge host / OPNsense / MGMT VLAN
 ↓
Cisco / OPNsense / servers / UPS / sensors / CCTV switches
```

Прямой проброс SSH/SNMP из интернета к судовым устройствам не предполагается.

---

## 3. Логическое разделение ролей

Очень важно разделять, что является источником каких данных.

```text
Baserow      = what should exist / inventory / intended state
Oxidized     = what configuration was actually saved from devices
Gitea        = history of configuration and project files
LibreNMS     = what is alive now / monitoring / observed state
AI Layer     = human-language interface over all sources
GitHub       = development workflow and task history
BCL          = production/self-hosted deployment target
```

Это разделение защищает проект от типичной ошибки: пытаться сделать один инструмент “для всего”.

---

## 4. High-Level Architecture

```text
                    ┌────────────────────────────┐
                    │          User / AI          │
                    │ ChatGPT / Codex / RAG       │
                    └─────────────┬──────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │          Baserow            │
                    │ Inventory / Source of Truth │
                    └─────────────┬──────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │        PostgreSQL           │
                    │   Structured data backend   │
                    └────────────────────────────┘

────────────────────────────────────────────────────────────────

                    ┌────────────────────────────┐
                    │           Gitea             │
                    │ Configs / Git history       │
                    └─────────────┬──────────────┘
                                  ▲
                                  │ Git commits
                                  │
                    ┌─────────────┴──────────────┐
                    │          Oxidized           │
                    │ Automated config collector  │
                    └─────────────┬──────────────┘
                                  │ SSH
                                  ▼
                    ┌────────────────────────────┐
                    │      Network Devices        │
                    │ Cisco / OPNsense / others   │
                    └────────────────────────────┘

────────────────────────────────────────────────────────────────

                    ┌────────────────────────────┐
                    │          LibreNMS           │
                    │ Monitoring / SNMP / Alerts  │
                    └─────────────┬──────────────┘
                                  │ SNMP / LLDP / CDP
                                  ▼
                    ┌────────────────────────────┐
                    │      Observed Network       │
                    │ Ports / links / traffic     │
                    └────────────────────────────┘
```

---

# 5. Компоненты стека

## 5.1 Baserow

### Роль

Baserow — это основной operational inventory interface.

Он должен заменить или постепенно вытеснить Google Sheets в задачах учёта сетевой инфраструктуры.

### Что хранится в Baserow

- список судов;
- сетевые устройства;
- IP-адреса;
- VLAN;
- switch ports;
- назначения портов;
- physical location оборудования;
- onboard comments;
- change notes;
- ответственные лица;
- ссылки на конфиги, схемы, мануалы и Git paths.

### Почему не NetBox

NetBox мощный, но для текущих задач проекта он может быть избыточен:

- высокий порог входа;
- сложная модель данных;
- много сущностей, которые не нужны на первом этапе;
- необходимость дисциплины enterprise-level.

Baserow выбран как более простой spreadsheet-like interface для ПКРЭ и shore engineering.

### Как Baserow взаимодействует с другими элементами

```text
User / ПКРЭ
 ↓ browser
Baserow
 ↓
PostgreSQL
 ↓ API
Scripts / AI / validation tools
```

В будущем данные из Baserow могут использоваться:

- для генерации inventory-файлов для Oxidized;
- для сверки planned port assignment vs running-config;
- для AI-запросов;
- для формирования отчетов по судам.

### Пример use-case

ПКРЭ на судне меняет подключение Patton:

```text
Old: Cisco Gi1/0/17
New: Cisco Gi1/0/20
```

Он открывает Baserow, меняет поле `Connected Device` у порта и добавляет комментарий.

Позже shore engineer или AI может спросить:

```text
Where is Patton connected on Fishing Tide?
```

Ответ должен строиться на данных Baserow и дополнительно проверяться по последнему config snapshot из Git.

---

## 5.2 PostgreSQL

### Роль

PostgreSQL — backend database для Baserow и потенциально будущих сервисов.

### Важное ограничение

PostgreSQL не должен использоваться onboard users напрямую.

Основные интерфейсы доступа:

- Baserow UI;
- Baserow API;
- controlled automation scripts;
- backup/restore procedures.

### Зачем сохранять PostgreSQL как отдельный элемент архитектуры

Даже если пользователи работают через Baserow, важно понимать, что данные не заперты внутри “black box”.

Возможные будущие сценарии:

- миграция данных;
- direct reporting;
- создание custom web interface;
- интеграция с AI/RAG;
- экспорт в CSV/JSON;
- автоматическая сверка с Git configs.

---

## 5.3 GitHub

### Роль

GitHub используется как development workflow platform.

Несмотря на то, что в production stack планируется Gitea, GitHub используется для:

- хранения исходной истории проекта;
- task-driven разработки;
- взаимодействия ChatGPT ↔ Codex;
- хранения `ai.workflow`;
- review по reports;
- последующего pull актуального состояния на BCL.

### Стандартный workflow

```text
1. ChatGPT создает TASK и Codex prompt
2. Пользователь копирует prompt в Codex
3. Codex выполняет изменения в GitHub repo
4. Codex создает report в ai.workflow/reports/
5. Пользователь сообщает: Codex закончил
6. ChatGPT сам читает report из GitHub
7. ChatGPT проверяет результат
8. После approval изменения подтягиваются на BCL
```

### GitHub vs Gitea

```text
GitHub = development workflow / project history / AI tasks
Gitea  = future self-hosted internal Git platform on BCL
```

На первом этапе GitHub является рабочей площадкой разработки, а Gitea появится позже как компонент развернутой платформы.

---

## 5.4 Gitea

### Роль

Gitea — будущая self-hosted Git platform внутри BCL.

Она должна хранить:

- конфиги сетевых устройств;
- backup-файлы;
- automation scripts;
- templates;
- документацию;
- внутренние технические заметки;
- возможно, зеркала рабочих репозиториев.

### Почему Gitea

Gitea рассматривается как lightweight self-hosted GitHub-like system.

Преимущества:

- работает локально на BCL;
- не требует тяжёлой DevOps-инфраструктуры;
- поддерживает web UI;
- позволяет хранить sensitive infrastructure history внутри контролируемой среды;
- хорошо подходит для Git-backed конфигов.

### Как Gitea взаимодействует с Oxidized

```text
Oxidized
 ↓ collects running-config
local Git repo
 ↓ commit
Gitea repository
```

Каждое изменение конфигурации устройства фиксируется как Git commit.

---

## 5.5 Oxidized

### Роль

Oxidized — automated configuration collector.

Он не управляет сетью и не пушит конфиги.

Его основная задача:

```text
читать конфиги устройств и сохранять историю изменений
```

### Что делает Oxidized

- подключается к устройству по SSH;
- выполняет команды получения конфигурации;
- сохраняет результат;
- сравнивает с предыдущей версией;
- создает Git commit при изменении.

Для Cisco IOS/XE типовой сценарий:

```text
SSH → show running-config → save → diff → Git commit
```

### Варианты развертывания

#### Вариант A — centralized collection

```text
BCL Oxidized
 ↓ VPN
Cisco on vessel
```

Плюсы:

- всё централизовано;
- проще администрировать;
- один Oxidized instance.

Минусы:

- зависит от WAN/VPN;
- если судно offline, сбор не выполнится.

#### Вариант B — local collector per vessel

```text
Ship edge host
 ↓ local MGMT VLAN
Cisco / OPNsense
 ↓ sync
central Git/Gitea
```

Плюсы:

- устойчивее к WAN instability;
- локальный сбор работает даже при плохом интернете;
- синхронизация может произойти позже.

Минусы:

- сложнее поддерживать;
- нужно ставить agent/service на каждое судно.

### Рекомендация для MVP

Начать с централизованного варианта:

```text
BCL Oxidized → VPN → 1 vessel → 1 Cisco switch
```

После успешного pilot можно решить, нужен ли local collector.

---

## 5.6 LibreNMS

### Роль

LibreNMS — monitoring and discovery platform.

Он показывает не “как должно быть”, а “что реально сейчас видно в сети”.

### Что мониторит LibreNMS

- availability устройств;
- interface up/down;
- traffic graphs;
- CRC/errors/discards;
- CPU/RAM;
- uptime;
- SNMP sensors;
- UPS battery status;
- port status;
- topology через LLDP/CDP.

### Как LibreNMS взаимодействует с остальной системой

```text
LibreNMS
 ↓ SNMP / LLDP / CDP / ARP
Network devices
```

LibreNMS может использоваться для сверки с Baserow:

```text
Baserow: Gi1/0/20 should be Patton
LibreNMS: Gi1/0/20 is down
Oxidized: running-config says access vlan 20
```

AI layer в будущем может собрать это в один ответ:

```text
По базе Patton должен быть на Gi1/0/20, порт сейчас down, конфигурация VLAN корректная. Возможная причина: устройство отключено физически или нет питания.
```

### Почему LibreNMS не первый этап

LibreNMS полезен, но он не наведёт порядок в inventory.

Поэтому сначала:

```text
Baserow → Git/Gitea → Oxidized → LibreNMS
```

---

## 5.7 VPN / Connectivity Layer

### Роль

VPN/overlay network обеспечивает безопасный доступ BCL к судовым MGMT-сегментам.

### Возможные варианты

- Tailscale;
- ZeroTier;
- WireGuard.

### Базовая схема

```text
BCL
 ↓ VPN Overlay
Onboard edge host / OPNsense
 ↓ routed access
MGMT VLAN
 ↓
Cisco / OPNsense / sensors / servers
```

### Требования

- не открывать SSH/SNMP наружу;
- ограничить доступ ACL;
- использовать отдельные service users;
- разделить management и user traffic;
- документировать per-vessel routing.

---

## 5.8 AI Layer

### Роль

AI Layer — future phase, который должен дать natural language interface поверх всех технических данных.

### Источники данных для AI

- Baserow inventory;
- Git/Gitea configs;
- Oxidized history;
- LibreNMS observed state;
- manuals;
- WhatsApp technical archive;
- network diagrams;
- NMEA schemes;
- project documentation;
- reports from `ai.workflow`.

### Примеры запросов

```text
Где сейчас подключен Patton на Fishing Tide?

Какие порты Cisco используются для CCTV на Fishing Force?

Что изменилось в конфиге C9200 за последние 30 дней?

Сгенерируй конфиг нового access port для VLAN 40.

Проверь, совпадает ли Baserow inventory с running-config.
```

### Важное ограничение

AI не должен быть первичным источником истины.

AI должен объяснять, искать, сопоставлять и помогать, но не заменять:

- Baserow inventory;
- Git config history;
- LibreNMS monitoring.

---

# 6. Основные потоки данных

## 6.1 Manual inventory update

```text
ПКРЭ / инженер
 ↓ browser
Baserow
 ↓
PostgreSQL
```

Используется для ручного обновления данных о портах, устройствах, IP, VLAN и комментариях.

---

## 6.2 Automated config backup

```text
Oxidized
 ↓ SSH over VPN
Cisco / OPNsense
 ↓ running-config
Git commit
 ↓
Gitea / Git repo
```

Используется для истории конфигураций.

---

## 6.3 Monitoring flow

```text
LibreNMS
 ↓ SNMP over VPN
Network devices
 ↓
Metrics / alerts / graphs
```

Используется для operational monitoring.

---

## 6.4 AI query flow, future

```text
User question
 ↓
AI Layer
 ↓
Baserow + Git configs + LibreNMS + docs
 ↓
Human-readable answer
```

---

# 7. Предлагаемая дорожная карта

## Phase 1 — Project Bootstrap

Цель:

- создать структуру репозитория;
- описать архитектуру;
- создать стандарт `ai.workflow`;
- подготовить базовые docs.

Сложность: Low.

---

## Phase 2 — Inventory MVP

Цель:

- развернуть PostgreSQL + Baserow;
- создать первую модель данных;
- импортировать одно pilot vessel;
- проверить удобство работы для shore engineering и ПКРЭ.

Pilot vessel:

```text
Fishing Tide
```

Сложность: Low-Medium.

---

## Phase 3 — Git/Gitea Foundation

Цель:

- подготовить Git structure для конфигов;
- определить naming convention;
- импортировать существующие Cisco/OPNsense configs;
- позже развернуть Gitea на BCL.

Сложность: Low-Medium.

---

## Phase 4 — Oxidized Pilot

Цель:

- подключить один Cisco switch;
- проверить SSH-доступ через VPN;
- сохранить первый running-config;
- проверить Git diff;
- подтвердить стабильность.

Сложность: Medium.

---

## Phase 5 — LibreNMS Pilot

Цель:

- подключить SNMP для одного судна;
- проверить discovery;
- получить interface graphs;
- настроить basic alerts.

Сложность: Medium-High.

---

## Phase 6 — AI/RAG Integration

Цель:

- индексировать inventory, configs, docs;
- реализовать query interface;
- добавить проверку соответствия inventory vs config;
- сформировать troubleshooting assistant.

Сложность: High.

---

# 8. Риски и ограничения

## 8.1 Overengineering

Главный риск — попытаться внедрить всё сразу.

Mitigation:

```text
Incremental deployment only.
```

---

## 8.2 Poor data discipline

Если onboard users не будут обновлять inventory, база устареет.

Mitigation:

- максимально простой UI;
- минимум обязательных полей;
- vessel-limited views;
- периодическая сверка с configs/LibreNMS;
- automation where possible.

---

## 8.3 Inconsistent vessel networks

На разных судах могут отличаться:

- IP plans;
- VLAN IDs;
- device names;
- MGMT access;
- VPN availability.

Mitigation:

- сначала pilot vessel;
- затем standard templates;
- потом постепенная унификация.

---

## 8.4 WAN instability

Судно может быть offline или иметь нестабильный канал.

Mitigation:

- tolerate failed polling;
- retry schedules;
- optional local collector in later phase;
- sync when online.

---

## 8.5 Credentials and secrets

Риск утечки паролей, SNMP communities, SSH credentials.

Mitigation:

- не хранить secrets в Git;
- использовать `.env.example`, но не `.env`;
- использовать dedicated service users;
- ограничивать доступ ACL;
- использовать VPN-only management.

---

# 9. Ожидаемый результат

После реализации проект должен дать:

- централизованный учёт fleet network infrastructure;
- понятный интерфейс для ПКРЭ;
- историю изменений конфигов;
- мониторинг состояния оборудования;
- основу для AI-assisted troubleshooting;
- снижение зависимости от одного Cursor-хоста;
- возможность разворачивать и поддерживать систему на BCL;
- единый GitHub workflow для разработки и контроля задач.

---

# 10. Current Status

Project bootstrap and architecture documentation phase.

Next planned step:

```text
TASK-001 — Project Bootstrap Structure
```
