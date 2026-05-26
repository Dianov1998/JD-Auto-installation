# scripts_vip — установка JD-сервера на VPS

Скрипты для первичной настройки машины, распаковки сборки, MySQL, authd и 32-bit библиотек.

| Скрипт | Назначение |
|--------|------------|
| **`install-jd.sh`** | Главное меню: LAMP, сборка, порты, база, authd |
| **`install-libs.sh`** | Зависимости `.so` для gs, gamedbd, gdeliveryd, glinkd |
| **`tools-jd.sh`** | Доп. инструменты на VPS после установки (см. раздел ниже) |

---

## Что получаете после установки

| Элемент | Путь / результат |
|---------|------------------|
| Игровая сборка | `/root/zxserver/` |
| Скрипт запуска | `/root/jd` (из архива сборки) |
| MySQL схема аккаунтов | база **`zx`** |
| Пароль MariaDB | `/root/.zxserver-install.cnf`, `/root/.my.cnf` |
| authd → MySQL | `/root/zxserver/authd/table.xml` (пароль подставляет install-jd) |
| База персов (gamedbd) | **`/db/gdb/`** (не в zxserver) |
| Логи logservice | **`/root/zxserver/logs1/`** |
| Логи процессов jd | `/root/zxserver/logs/` |
| Запуск ядер (.so) | `/root/zxserver/env-libs.sh` (после install-libs) |
| Бэкап | `/root/back.sh` → архивы в `/backup/` (настраивается **tools-jd.sh**) |

---



Рекомендуемый порядок пунктов меню:

```text
1 → LAMP (Apache + MariaDB + phpMyAdmin), запомнить пароль MySQL
2 → Сборка (4.8.0 на CentOS), база zx, authd, библиотеки
4 → Открыть порты firewalld (если CentOS)
./jd start
```

---

## install-jd.sh — меню

Запуск: **`./install-jd.sh`** (только **root**).  
При старте скрипт может предложить обновление с `jd.legend-it.ru` (`UPDATE_MODE=ask|auto|off`).

| Пункт | Действие |
|-------|----------|
| **1** | **LAMP**: Apache, MariaDB/MySQL, phpMyAdmin, PHP; пароль root MySQL → в `.zxserver-install.cnf` |
| **2** | **Сборка**: скачать `build-*.tar.gz` → распаковка в `/` → база **zx** → `table.xml` → **install-libs.sh** |
| **3** | Доп. инструменты (внешний tools-скрипт) |
| **4** | **Firewall**: открыть/закрыть порты (CentOS: firewalld) |
| **5** | Только **install-libs.sh** (без переустановки сборки) |
| **6** | Только **база zx + authd** (чистая / с ЛК Алекса), сборку не трогает |
| **7** | Выход |

### П.1 — CentOS (кратко)

- MariaDB, Apache (httpd), PHP 7.4, phpMyAdmin  
- **Выбор Java:** `1` = Java 7 (сборки до 4.6.0), `2` = Java 8 (4.8.0)  
- Лимиты загрузки SQL в phpMyAdmin (~200M)  
- Firewalld: порты по выбору  
- Пароль сохраняется в **`/root/.zxserver-install.cnf`**

### П.2 — Установка сборки (4 шага)

1. **Архив** — скачивание и распаковка (CentOS: **4.8.0** → `/root/zxserver`, `/root/jd`, …)  
2. **MySQL `zx`** — выбор дампа:  
   - `1` — **zx.sql** (чистая база)  
   - `2` — **zx_LK.sql** (с личным кабинетом Алекса)  
   - `n` — пропуск импорта  
3. **authd** — пароль MariaDB записывается в **`authd/table.xml`** (не шаблон `qweKEQXrqal`)  
4. **Библиотеки** — автоматически вызывается **`install-libs.sh`**

Поддерживаемые сборки (CentOS 7, URL в `install-jd.sh`):

| Версия | Java в п.1 |
|--------|------------|
| 3.1.1, 4.2.0, 4.4.0, 4.6.0 | **7** (`java-1.7.0-openjdk`) |
| 4.8.0 | **8** (`java-1.8.0-openjdk`) |

Ubuntu в установщике **не поддерживается** — только CentOS.

### П.6 — только база и authd

Если сборка уже стоит, а нужно перезалить SQL или обновить пароль в `table.xml` / `back.sh`:

```bash
./install-jd.sh   →   6
```

Нужен рабочий пароль в `/root/.zxserver-install.cnf` (из п.1).

---

## install-libs.sh — библиотеки 32-bit

Вызывается из **п.2** и **п.5**, или отдельно:

```bash
cd /root              # каталог со скриптами на VPS (часто просто /root)
./install-libs.sh
```

### Что делает (по шагам)

1. Удаляет старый **`/etc/ld.so.conf.d/zxserver.conf`** (глобальный ldconfig ломает yum/bash)  
2. Проверяет **gs, gamedbd, gdeliveryd, glinkd, authd** через `ldd` / objdump  
3. Ставит пакеты (**glibc.i686**, **libstdc++.i686**, …) из yum/apt  
4. Финальная проверка  
5. Создаёт **`/root/zxserver/env-libs.sh`** — `LD_LIBRARY_PATH` только для запуска ядер  

**Важно:** не добавляйте `zxserver/lib` в глобальный `ldconfig`. Для ручного запуска:

```bash
source /root/zxserver/env-libs.sh
cd /root/zxserver/gamed && ./gs ...
```

### Флаги

| Команда | Действие |
|---------|----------|
| `./install-libs.sh` | Полный цикл: проверка → yum → проверка |
| `./install-libs.sh --check` | Только просмотр ✓/✗, без установки |
| `./install-libs.sh --verify` | Только финальная проверка ldd |
| `./install-libs.sh --no-install` | Анализ без yum |
| `./install-libs.sh --ask` | Спросить перед yum |
| `./install-libs.sh --help` | Справка |

Доп. карта пакетов: **`libs-packages.map`** (рядом со скриптом или с `jd.legend-it.ru`).

Переменные:

```bash
ZXSERVER_ROOT=/root/zxserver ./install-libs.sh
```

---

## tools-jd.sh — доп. инструменты на VPS

Скрипт для **CentOS / RHEL** (Ubuntu не поддерживается). Запуск только от **root**, после установки сервера через **install-jd.sh**.

Также вызывается из **install-jd.sh → п.3** («Доп. инструменты»).

```bash
cd /root
chmod +x tools-jd.sh
./tools-jd.sh
```

Скрипты обычно лежат прямо в **`/root`** (рядом с `install-jd.sh`, `install-libs.sh`).  
**install-jd.sh → п.3** сам скачивает `tools-jd.sh` в **текущий каталог** и запускает его — отдельный `cd` не нужен, если вы уже в `/root`.

При старте проверяется CentOS; открывается интерактивное меню.

### Главное меню

| Пункт | Название | Что делает |
|-------|----------|------------|
| **1** | Firewall | Показывает состояние **firewalld** (`firewall-cmd --state`, `--list-all`) |
| **2** | Backup сервера | Ставит `/root/back.sh`, cron, опционально Git в каталоге бэкапов |
| **3** | Стартовые персонажи | `gamedbd … exportclsconfig` — перезапись шаблонов персов |
| **4** | Очистка кэша ОС | Сброс RAM-кэша и/или `yum clean`; можно повесить на cron |
| **5** | Обновление топа | `toplist` + cron, первый запуск сразу |
| **6** | Выход | Закрыть меню |

---

### П.1 — Статус Firewall

Только просмотр: работает ли firewalld и какие зоны/порты открыты.  
Открытие портов по-прежнему через **install-jd.sh → п.4**.

---

### П.2 — Настройка backup сервера

Разворачивает авто-бэкап на основе шаблона `sborki/.../back.sh` → **`/root/back.sh`**.

**Что попадает в архив `backup-ДАТА.tar`:**

- дамп MySQL **`zx`**;
- копия **`/db/`** (база персонажей gamedbd);
- архивы старше **30 дней** в каталоге бэкапов удаляются.

| Шаг мастера | Вопрос | Варианты |
|-------------|--------|----------|
| Режим | Куда сохранять | **1** — только VPS; **2** — VPS + push в Git |
| Каталог | Путь архивов | по умолчанию **`/backup`** |
| Частота | Cron | каждые **5 мин** / **30 мин** / **2 ч** / **раз в сутки (00:00)** |
| Git (режим 2) | Репозиторий | URL, ветка, HTTPS+token или SSH, `user.name` / `email` |

Пароль MySQL подставляется из **`/root/.zxserver-install.cnf`**, если файл есть (после install-jd п.1/2).

**Режим «только VPS»:** в `back.sh` остаётся `BACKUP_ENABLE_GIT=0` — выполняется только создание `.tar`, без git.

**Режим «VPS + Git»:** `BACKUP_ENABLE_GIT=1`, в каталоге бэкапов (`/backup`) инициализируется git; после каждого cron — `add` / `commit` / `push`.

**Подготовка GitHub (режим 2):**

1. Создать **пустой** репозиторий (без README).
2. Personal Access Token с правом **`repo`** (или SSH-ключ в GitHub).
3. В мастере указать URL, ветку (`master` / `main`), логин и token.

| Файл / лог | Назначение |
|------------|------------|
| `/root/back.sh` | Скрипт бэкапа |
| `/var/log/jd-backup.log` | Лог cron |
| `crontab -l` | Строка с меткой `# jd-auto-backup` |

Проверка вручную: `/root/back.sh`

---

### П.3 — Перезапись стартовых персонажей

Выполняет:

```bash
cd /root/zxserver/gamedbd
./gamedbd gamesys.conf exportclsconfig
```

**Важно:** если вы редактировали персонажей на аккаунтах **16, 32, 48**, они будут **перезаписаны**. Новые персонажи получат все значения от **стартовых** персонажей сборки.

Перед запуском скрипт просит ввести **`yes`**. Рекомендуется остановить сервер: **`./jd stop`**.

---

### П.4 — Очистка кэша ОС

Освобождает RAM, занятую кэшем диска (`sync` + `echo 3 > /proc/sys/vm/drop_caches`). Файлы на диске **не удаляются**. Полезно при нехватке памяти на слабом VPS.

| Подпункт | Действие |
|----------|----------|
| **1** | Сейчас: только сброс RAM-кэша |
| **2** | Сейчас: RAM + **`yum clean all`** |
| **3** | Авто **cron** раз в сутки (**04:00**) → подменю **a)** только RAM, **b)** RAM + yum |
| **0** | Отмена |

Для пунктов 1–2 и установки cron нужно подтверждение **`yes`**.

| Файл / лог | Назначение |
|------------|------------|
| `/root/jd-clear-cache.sh` | Скрипт для cron (создаётся при п.4 → 3) |
| `/var/log/jd-cache-clear.log` | Лог авто-очистки |
| `crontab -l` | Строка с меткой `# jd-auto-cache-clear` |

Повторная настройка cron заменяет предыдущую задачу очистки. Ручной запуск: `/root/jd-clear-cache.sh`

---

### П.5 — Авто-обновление топа (toplist)

Создаёт **`/root/top.sh`** и ставит **cron** (5 мин / 30 мин / 2 ч / раз в сутки).

Пути к базам **не вводятся вручную**: скрипт читает **`homedir`** из секции **`[storage]`** в трёх файлах:

| Файл | Путь (Пример)|
|------|----------------------|
| `gamedbd/gamesys.conf` | `/db_server/gdb/dbhome` |
| `gamedbd/gamesyslj.conf` | `/db_server/gdb/lingjingdbhomewdb` |
| `gamedbd/isgamesys.conf` | `/db_server/gdb/interdbhomewdb` |

Для каждого найденного пути в `top.sh` добавляется своя строка `./toplist …`. Дубликаты путей объединяются.

**Если файла нет** (например на **3.1.1** нет `gamesyslj.conf`) — строка вроде `gamesyslj.conf — нет файла (в этой сборке не используется), пропуск`, ошибки нет. Достаточно **хотя бы одного** конфига с `homedir` (обычно `gamesys.conf` + при наличии `isgamesys.conf`).

| Шаг | Вопрос | По умолчанию |
|-----|--------|--------------|
| Каталог | Где лежит `toplist` | `/root/zxserver/toplist` |
| Частота | Cron | 1–4 (как в п.2) |

После **`yes`**: `top.sh` → cron → **первое обновление всех баз сразу**. Лог: **`/var/log/jd-toplist.log`** (в консоль toplist не пишет).

| Файл / лог | Назначение |
|------------|------------|
| `/root/top.sh` | Запуск toplist |
| `/var/log/jd-toplist.log` | Лог обновлений |
| `crontab -l` | Строка `# jd-auto-toplist` |

---

### Что создаёт tools-jd.sh на сервере

```text
/root/back.sh              бэкап (п.2)
/root/top.sh               обновление топа (п.5)
/root/jd-clear-cache.sh    очистка кэша по cron (п.4 → 3)
/backup/                   каталог архивов (по умолчанию, п.2)
/backup/.git/              только при режиме Git
/var/log/jd-backup.log
/var/log/jd-toplist.log
/var/log/jd-cache-clear.log
crontab root               # jd-auto-backup, # jd-auto-toplist, # jd-auto-cache-clear
```

---

## Важные пути (не путать)

```text
/root/zxserver/              сборка, бинарники, config
/root/zxserver/gamed/config/ .data, lua, txt
/root/zxserver/logs1/        логи logservice (world2.*)
/root/zxserver/logs/         логи jd (logservice.log, gamedbd.log…)

/db/gdb/dbhomewdb/dbdata/    база персов gamedbd  ← бэкап /db/
/db/gdb/backup/              бэкапы gamedbd

MySQL аккаунты               база zx (mysqldump)
/root/.zxserver-install.cnf  пароль MariaDB для п.2/6
/root/zxserver/authd/table.xml
```

Бэкап и cron настраиваются через **`tools-jd.sh` → п.2** (см. раздел выше). Пароль в `/root/back.sh` обновляет **install-jd** (п.2/6).

---

## Типовые сценарии

### Новый сервер CentOS 7

```bash
./install-jd.sh    # 1 (Java 7 или 8) → 2 (версия сборки) → 4
./jd stop          # если что-то уже запущено
./jd start
```

### Перезалить только SQL / пароль authd

```bash
./install-jd.sh    # 6
```

### После обновления файлов в /root/zxserver

```bash
./install-libs.sh --verify
./jd restart
```

### Бэкап, стартовые персы, очистка RAM

```bash
cd /root && ./tools-jd.sh    # 2 backup, 3 персы, 4 кэш, 5 топ
```

## Частые проблемы

| Симптом | Решение |
|---------|---------|
| «Пароль qweKEQXrqal» / MariaDB не пускает | П.1 заново или вручную: правильный пароль в `/root/.zxserver-install.cnf` |
| `ldd ./gs` — not found | `./install-libs.sh` или п.5; на CentOS: `yum install glibc.i686 libstdc++.i686` |
| yum/bash сломался после старых скриптов | `./install-libs.sh` (удалит zxserver из ldconfig) |
| П.2: база не импортировалась | Сначала п.1; проверить `mysql -u root -p` |
| phpMyAdmin не грузит большой SQL | П.1 уже поднимает лимиты; перезапустить httpd |

---


Установка = **install-jd** + **install-libs**. Обслуживание VPS = **tools-jd.sh**. Запуск игры = **`./jd start`** / **`./jd restart`**.
