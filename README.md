# scripts_vip — установка JD-сервера на VPS

Скрипты для первичной настройки машины, распаковки сборки, MySQL, authd и 32-bit библиотек.

| Скрипт | Назначение |
|--------|------------|
| **`install-jd.sh`** | Главное меню: LAMP, сборка, порты, база, authd |
| **`install-libs.sh`** | Зависимости `.so` для gs, gamedbd, gdeliveryd, glinkd |

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
| Бэкап (если есть gitback) | `/root/gitback.sh` → архивы в `/root/epsilonjd/` |

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
- Лимиты загрузки SQL в phpMyAdmin (~200M)  
- Firewalld: базовые правила  
- Пароль сохраняется в **`/root/.zxserver-install.cnf`**

### П.2 — Установка сборки (4 шага)

1. **Архив** — скачивание и распаковка (CentOS: **4.8.0** → `/root/zxserver`, `/root/jd`, …)  
2. **MySQL `zx`** — выбор дампа:  
   - `1` — **zx.sql** (чистая база)  
   - `2` — **zx_LK.sql** (с личным кабинетом Алекса)  
   - `n` — пропуск импорта  
3. **authd** — пароль MariaDB записывается в **`authd/table.xml`** (не шаблон `qweKEQXrqal`)  
4. **Библиотеки** — автоматически вызывается **`install-libs.sh`**

Поддерживаемые сборки (URL в начале `install-jd.sh`):

| ОС | Версии |
|----|--------|
| CentOS | 4.8.0 |
| Ubuntu | 3.1.1, 4.2.0, 4.4.0, 4.6.0 |

### П.6 — только база и authd

Если сборка уже стоит, а нужно перезалить SQL или обновить пароль в `table.xml` / `gitback.sh`:

```bash
./install-jd.sh   →   6
```

Нужен рабочий пароль в `/root/.zxserver-install.cnf` (из п.1).

---

## install-libs.sh — библиотеки 32-bit

Вызывается из **п.2** и **п.5**, или отдельно:

```bash
cd /root/scripts_vip   # или каталог со скриптом
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

Пример бэкапа **`gitback.sh`**: копирует **`/db/`** + дамп **`zx`** → `/root/epsilonjd/backup-*.tar`.  
Пароль в `gitback.sh` install-jd может обновить при п.2/6.

---

## Типовые сценарии

### Новый сервер CentOS 7

```bash
./install-jd.sh    # 1 → 2 → 4
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

## Частые проблемы

| Симптом | Решение |
|---------|---------|
| «Пароль qweKEQXrqal» / MariaDB не пускает | П.1 заново или вручную: правильный пароль в `/root/.zxserver-install.cnf` |
| `ldd ./gs` — not found | `./install-libs.sh` или п.5; на CentOS: `yum install glibc.i686 libstdc++.i686` |
| yum/bash сломался после старых скриптов | `./install-libs.sh` (удалит zxserver из ldconfig) |
| П.2: база не импортировалась | Сначала п.1; проверить `mysql -u root -p` |
| phpMyAdmin не грузит большой SQL | П.1 уже поднимает лимиты; перезапустить httpd |

---


Установка = **install-jd** + **install-libs**. Запуск = **`./jd start`** / **`./jd restart`**.
