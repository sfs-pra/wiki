# Руководство по использованию `systemd-sysext` в Arch Linux и Debian

## Содержание

- [1. Принципы работы `systemd-sysext`](#1-принципы-работы-systemd-sysext)
  - [Основные понятия](#основные-понятия)
  - [Требования к структуре образа](#требования-к-структуре-образа)
- [2. Совместимость версий и параметр `SYSEXT_LEVEL`](#2-совместимость-версий-и-параметр-sysext_level)
  - [Как `systemd-sysext` проверяет совместимость](#как-systemd-sysext-проверяет-совместимость)
  - [Особенности в Arch Linux (`SYSEXT_LEVEL=1`)](#особенности-в-arch-linux-sysext_level1)
  - [Особенности в Debian (`VERSION_ID`)](#особенности-в-debian-version_id)
  - [Универсальный вариант (`ID=_any`)](#универсальный-вариант-id_any)
- [3. Базовый рабочий процесс (ручная сборка)](#3-базовый-рабочий-процесс-ручная-сборка)
  - [Подготовка структуры и метаданных](#подготовка-структуры-и-метаданных)
  - [Сборка SquashFS-образа](#сборка-squashfs-образа)
  - [Подключение и отключение](#подключение-и-отключение)
- [4. Упаковка GUI-приложений (на примере `galculator`)](#4-упаковка-gui-приложений-на-примере-galculator)
  - [Особенности GUI-модулей](#особенности-gui-модулей)
  - [Автоматическая сборка для Arch Linux (`build-sysext-arch.sh`)](#автоматическая-сборка-для-arch-linux-build-sysext-archsh)
  - [Автоматическая сборка для Debian (`build-sysext-debian.sh`)](#автоматическая-сборка-для-debian-build-sysext-debiansh)
- [5. Удобный скрипт-переключатель (`sysext-toggle`)](#5-удобный-скрипт-переключатель-sysext-toggle)
  - [Код скрипта](#код-скрипта)
  - [Примеры использования](#примеры-использования)
- [6. Настройка автодополнения (`Tab`)](#6-настройка-автодополнения-tab)
  - [Для Bash](#для-bash)
  - [Для Zsh](#для-zsh)
- [7. Важные нюансы и ограничения](#7-важные-нюансы-и-ограничения)

---

## 1. Принципы работы `systemd-sysext`

**`systemd-sysext` (System Extension Images)** — это механизм в составе `systemd`, предназначенный для динамического расширения неизменяемых (immutable) или стандартных систем без перезагрузки и модификации базовой файловой системы.

### Основные понятия
* **Форматы образов:** Поддерживаются raw-образы дисков (GPT, опционально с dm-verity), чистые образы файловых систем **SquashFS** и **EROFS** (обычно с расширением `.raw`), а также обычные директории.
* **Точки монтирования:** Образы накладываются поверх `/usr` и `/opt` с помощью **OverlayFS** в режиме «только для чтения» (`read-only`). Для аналогичной работы с `/etc` используется утилита `systemd-confext`.
* **Директории поиска:** По умолчанию `systemd-sysext` ищет расширения в `/etc/extensions/`, `/run/extensions/` и `/var/lib/extensions/`.
* **Динамическое управление:** Команда `systemd-sysext refresh` на лету пересобирает слои OverlayFS при добавлении или удалении образов.

### Требования к структуре образа
1. **Иерархия каталогов:** Файлы внутри образа должны находиться в директориях `/usr/...` или `/opt/...` (корень образа соответствует корню системы `/`).
2. **Файл метаданных:** Внутри образа обязательно должен присутствовать файл `/usr/lib/extension-release.d/extension-release.<ИМЯ_ОБРАЗА>`, где `<ИМЯ_ОБРАЗА>` строго совпадает с именем `.raw`-файла без расширения.
3. **Права доступа:** Владельцем образа `.raw` и директории `/var/lib/extensions` должен быть `root:root`, а права на запись для других пользователей должны отсутствовать (`chmod 644`).

---

## 2. Совместимость версий и параметр `SYSEXT_LEVEL`

### Как `systemd-sysext` проверяет совместимость
Перед подключением образа `systemd-sysext` сверяет файл метаданных `extension-release.<ИМЯ>` внутри образа с файлом `/etc/os-release` хост-системы. Это защищает систему от подключения бинарно несовместимых библиотек от другой ОС или другого релиза.

### Особенности в Arch Linux (`SYSEXT_LEVEL=1`)
Arch Linux использует модель rolling release, поэтому в его `/etc/os-release` **отсутствует поле `VERSION_ID`**. 

Если указать в `extension-release` только `ID=arch`, утилита `systemd-sysext` может отклонить образ из-за отсутствия привязки к версии/уровню совместимости. Чтобы явно обозначить уровень совместимости, используется параметр `SYSEXT_LEVEL`:

```ini
ID=arch
SYSEXT_LEVEL=1
```

**Как это работает:**
* `SYSEXT_LEVEL` — это произвольный строковый или числовой идентификатор уровня совместимости (например, `1`, `2` или дата сборки `2026.11`).
* Если в `/etc/os-release` хоста не задан свой `SYSEXT_LEVEL`, `systemd-sysext` принимает любой образ с совпадающим `ID=arch` и наличием поля `SYSEXT_LEVEL`.
* Если же вы вручную добавите `SYSEXT_LEVEL=2` в `/etc/os-release` хоста, система начнёт принимать только те модули, у которых указан `SYSEXT_LEVEL=2`, отсеивая устаревшие сборки.

### Особенности в Debian (`VERSION_ID`)
Debian имеет фиксированные релизы, и в его `/etc/os-release` всегда присутствуют оба поля (например, `ID=debian` и `VERSION_ID="12"` или `"13"`). Поэтому для Debian в файле метаданных достаточно указать точное совпадение с хостом:

```ini
ID=debian
VERSION_ID=12
```

### Универсальный вариант (`ID=_any`)
Если модуль содержит полностью статически слинкованные бинарники (на Go, Rust или AppImage), которые не зависят от системных библиотек (`glibc`, `openssl` и др.), проверку дистрибутива можно отключить:

```ini
ID=_any
```

---

## 3. Базовый рабочий процесс (ручная сборка)

### Подготовка структуры и метаданных
Создайте рабочую директорию и поместите туда нужные файлы:

```bash
mkdir -p mytools/usr/bin
mkdir -p mytools/usr/lib/extension-release.d

# Поместите ваши бинарники в mytools/usr/bin/

# Создайте файл метаданных (для Arch Linux)
cat << 'EOF' > mytools/usr/lib/extension-release.d/extension-release.mytools
ID=arch
SYSEXT_LEVEL=1
EOF
```

### Сборка SquashFS-образа
Установите пакет `squashfs-tools` (`pacman -S squashfs-tools` в Arch или `apt install squashfs-tools` в Debian) и соберите образ:

```bash
mksquashfs mytools mytools.raw -noappend -comp zstd -all-root
```
*(Флаг `-all-root` автоматически назначает владельцем всех файлов внутри образа `root:root`).*

### Подключение и отключение

```bash
# Подключение
sudo mkdir -p /var/lib/extensions
sudo mv mytools.raw /var/lib/extensions/
sudo systemctl enable --now systemd-sysext.service
sudo systemd-sysext refresh

# Проверка статуса
systemd-sysext status

# Временное отключение конкретного модуля
sudo mv /var/lib/extensions/mytools.raw /var/lib/extensions/mytools.raw.disabled
sudo systemd-sysext refresh

# Полное отключение всех модулей
sudo systemd-sysext unmerge
```

---

## 4. Упаковка GUI-приложений (на примере `galculator`)

### Особенности GUI-модулей
Чтобы графическое приложение появилось в меню рабочего стола (GNOME, KDE, XFCE и др.) сразу после подключения `.raw`-образа без перезагрузки сессии, необходимо:
1. Убедиться, что внутри образа присутствуют ярлык `/usr/share/applications/*.desktop` и иконки в `/usr/share/icons/hicolor/...`.
2. После вызова `systemd-sysext refresh` обновить системные кэши ярлыков и иконок:
   ```bash
   sudo update-desktop-database /usr/share/applications
   sudo gtk-update-icon-cache -f -t /usr/share/icons/hicolor
   ```

В качестве примера ниже используется лёгкий инженерный калькулятор **`galculator`** — он не требует прав `root` или настройки Polkit и запускается от обычного пользователя сразу после подключения модуля.

---

### Автоматическая сборка для Arch Linux (`build-sysext-arch.sh`)

Скрипт скачивает пакет и **только те зависимости, которых ещё нет** в вашей хост-системе, распаковывает их и собирает `.raw`-образ.

Сохраните как `build-sysext-arch.sh` и выполните `chmod +x build-sysext-arch.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PKG_NAME="${1:-galculator}"
EXT_NAME="${2:-$PKG_NAME}"
WORK_DIR=$(mktemp -d -t sysext-arch-XXXXXX)

cleanup() {
    sudo rm -rf "$WORK_DIR"
}
trap cleanup EXIT

for cmd in pacman mksquashfs tar; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "Ошибка: не найдена утилита '$cmd'. Установите squashfs-tools."
        exit 1
    fi
done

mkdir -p "$WORK_DIR/pkgs" "$WORK_DIR/rootfs"

echo "==> Скачивание $PKG_NAME и недостающих в системе зависимостей..."
sudo pacman -Syw --cachedir "$WORK_DIR/pkgs" --noconfirm "$PKG_NAME"

echo "==> Распаковка пакетов в rootfs..."
for pkg in "$WORK_DIR"/pkgs/*.pkg.tar*; do
    [ -e "$pkg" ] || continue
    echo "    Распаковка $(basename "$pkg")..."
    sudo tar -xf "$pkg" -C "$WORK_DIR/rootfs"         --exclude='.BUILDINFO'         --exclude='.MTREE'         --exclude='.PKGINFO'         --exclude='.INSTALL'
done

echo "==> Очистка директорий вне /usr и /opt..."
sudo find "$WORK_DIR/rootfs" -mindepth 1 -maxdepth 1 ! -name 'usr' ! -name 'opt' -exec rm -rf {} +

echo "==> Создание метаданных extension-release..."
sudo mkdir -p "$WORK_DIR/rootfs/usr/lib/extension-release.d"
sudo tee "$WORK_DIR/rootfs/usr/lib/extension-release.d/extension-release.${EXT_NAME}" > /dev/null <<EOF
ID=arch
SYSEXT_LEVEL=1
EOF

OUTPUT_FILE="${EXT_NAME}.raw"
echo "==> Сборка SquashFS-образа: ${OUTPUT_FILE}..."
sudo mksquashfs "$WORK_DIR/rootfs" "$OUTPUT_FILE" -noappend -comp zstd -all-root
sudo chown "$USER:$USER" "$OUTPUT_FILE"

echo "==> Готово! Образ сохранён как ${OUTPUT_FILE}"
```

**Запуск:**
```bash
./build-sysext-arch.sh galculator
```

---

### Автоматическая сборка для Debian (`build-sysext-debian.sh`)

В Debian для скачивания пакета вместе с отсутствующими зависимостями используется `apt-get install --download-only`, а распаковка `.deb`-архивов выполняется через `dpkg-deb -x`. Также скрипт автоматически подставляет `ID` и `VERSION_ID` из вашего `/etc/os-release`.

Сохраните как `build-sysext-debian.sh` и выполните `chmod +x build-sysext-debian.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PKG_NAME="${1:-galculator}"
EXT_NAME="${2:-$PKG_NAME}"
WORK_DIR=$(mktemp -d -t sysext-deb-XXXXXX)

cleanup() {
    sudo rm -rf "$WORK_DIR"
}
trap cleanup EXIT

for cmd in apt-get dpkg-deb mksquashfs; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "Ошибка: не найдена утилита '$cmd'. Установите squashfs-tools."
        exit 1
    fi
done

mkdir -p "$WORK_DIR/pkgs" "$WORK_DIR/rootfs"

echo "==> Обновление индексов и скачивание $PKG_NAME с недостающими зависимостями..."
sudo apt-get update -qq
sudo apt-get install --download-only --yes     -o Dir::Cache::archives="$WORK_DIR/pkgs"     "$PKG_NAME"

echo "==> Распаковка .deb пакетов в rootfs..."
for deb in "$WORK_DIR"/pkgs/*.deb; do
    [ -e "$deb" ] || continue
    echo "    Распаковка $(basename "$deb")..."
    sudo dpkg-deb -x "$deb" "$WORK_DIR/rootfs"
done

# В современных версиях Debian (12+) действует usrmerge (/bin -> /usr/bin, /lib -> /usr/lib).
# Если старый пакет содержит файлы в корневых /bin или /lib, переносим их в /usr.
for dir in bin sbin lib lib64; do
    if [[ -d "$WORK_DIR/rootfs/$dir" && ! -L "$WORK_DIR/rootfs/$dir" ]]; then
        sudo mkdir -p "$WORK_DIR/rootfs/usr/$dir"
        sudo cp -a "$WORK_DIR/rootfs/$dir/." "$WORK_DIR/rootfs/usr/$dir/"
        sudo rm -rf "$WORK_DIR/rootfs/$dir"
    fi
done

echo "==> Очистка директорий вне /usr и /opt..."
sudo find "$WORK_DIR/rootfs" -mindepth 1 -maxdepth 1 ! -name 'usr' ! -name 'opt' -exec rm -rf {} +

# Считываем ID и VERSION_ID из /etc/os-release текущей системы
OS_ID=$(. /etc/os-release && echo "${ID:-debian}")
OS_VER=$(. /etc/os-release && echo "${VERSION_ID:-}")

echo "==> Создание метаданных extension-release (ID=$OS_ID, VERSION_ID=$OS_VER)..."
sudo mkdir -p "$WORK_DIR/rootfs/usr/lib/extension-release.d"
{
    echo "ID=${OS_ID}"
    [[ -n "$OS_VER" ]] && echo "VERSION_ID=${OS_VER}"
} | sudo tee "$WORK_DIR/rootfs/usr/lib/extension-release.d/extension-release.${EXT_NAME}" > /dev/null

OUTPUT_FILE="${EXT_NAME}.raw"
echo "==> Сборка SquashFS-образа: ${OUTPUT_FILE}..."
sudo mksquashfs "$WORK_DIR/rootfs" "$OUTPUT_FILE" -noappend -comp zstd -all-root
sudo chown "$USER:$USER" "$OUTPUT_FILE"

echo "==> Готово! Образ сохранён как ${OUTPUT_FILE}"
```

**Запуск:**
```bash
./build-sysext-debian.sh galculator
```

---

## 5. Удобный скрипт-переключатель (`sysext-toggle`)

Этот универсальный скрипт (подходит и для Arch Linux, и для Debian) позволяет одной командой включать или отключать выбранный `.raw` модуль и автоматически обновляет кэш иконок и меню рабочего стола.

### Код скрипта
Сохраните код в `/usr/local/bin/sysext-toggle` и сделайте исполняемым (`sudo chmod +x /usr/local/bin/sysext-toggle`):

```bash
#!/usr/bin/env bash
set -euo pipefail

EXT_DIR="/var/lib/extensions"

if [[ $EUID -ne 0 ]]; then
    exec sudo "$0" "$@"
fi

mkdir -p "$EXT_DIR"

update_desktop_caches() {
    echo "==> Обновление кэша меню и иконок..."
    if command -v update-desktop-database &>/dev/null; then
        update-desktop-database /usr/share/applications 2>/dev/null || true
    fi
    if command -v gtk-update-icon-cache &>/dev/null; then
        gtk-update-icon-cache -f -t /usr/share/icons/hicolor 2>/dev/null || true
    fi
    systemctl daemon-reload
}

show_status() {
    echo "=== Доступные модули в $EXT_DIR ==="
    shopt -s nullglob
    local found=0
    for f in "$EXT_DIR"/*.raw "$EXT_DIR"/*.raw.disabled; do
        found=1
        local base
        base=$(basename "$f")
        if [[ "$base" == *.raw ]]; then
            echo "  [ВКЛ]  ${base%.raw}"
        else
            echo "  [ВЫКЛ] ${base%.raw.disabled}"
        fi
    done
    if [[ $found -eq 0 ]]; then
        echo "  (модули не найдены)"
    fi
    echo ""
    systemd-sysext status
}

toggle_module() {
    local name="$1"
    local active_file="$EXT_DIR/${name}.raw"
    local disabled_file="$EXT_DIR/${name}.raw.disabled"

    if [[ -f "$active_file" ]]; then
        echo "==> Отключение модуля: $name"
        mv "$active_file" "$disabled_file"
        systemd-sysext refresh
        update_desktop_caches
        echo "==> Модуль '$name' отключен."
    elif [[ -f "$disabled_file" ]]; then
        echo "==> Включение модуля: $name"
        mv "$disabled_file" "$active_file"
        systemd-sysext refresh
        update_desktop_caches
        echo "==> Модуль '$name' подключен."
    else
        echo "Ошибка: модуль '$name' не найден в $EXT_DIR"
        exit 1
    fi
}

case "${1:-status}" in
    status|list)
        show_status
        ;;
    off-all)
        echo "==> Отмонтирование всех sysext-слоёв..."
        systemd-sysext unmerge
        update_desktop_caches
        ;;
    on-all)
        echo "==> Монтирование всех активных sysext-слоёв..."
        systemd-sysext refresh
        update_desktop_caches
        ;;
    *)
        toggle_module "$1"
        ;;
esac
```

### Примеры использования

```bash
# Скопировать собранный образ в каталог расширений
sudo cp galculator.raw /var/lib/extensions/

# Включить или отключить модуль galculator (переключатель)
sysext-toggle galculator

# Показать список всех модулей ([ВКЛ] / [ВЫКЛ]) и статус OverlayFS
sysext-toggle

# Временно отмонтировать вообще все активные расширения (перед обновлением системы)
sysext-toggle off-all

# Вернуть все активные расширения обратно
sysext-toggle on-all
```

---

## 6. Настройка автодополнения (`Tab`)

Чтобы имена модулей из `/var/lib/extensions` и команды скрипта подставлялись автоматически при нажатии `Tab`, установите конфигурацию для вашей оболочки.

### Для Bash
```bash
sudo mkdir -p /usr/share/bash-completion/completions
sudo tee /usr/share/bash-completion/completions/sysext-toggle > /dev/null << 'EOF'
_sysext_toggle_completions() {
    local cur commands modules
    cur="${COMP_WORDS[COMP_CWORD]}"

    if [[ $COMP_CWORD -eq 1 ]]; then
        commands="status list off-all on-all"
        if [[ -d /var/lib/extensions ]]; then
            modules=$(find /var/lib/extensions -maxdepth 1 \( -name "*.raw" -o -name "*.raw.disabled" \) -printf "%f
" 2>/dev/null | sed -E 's/\.raw(\.disabled)?$//')
        fi
        COMPREPLY=( $(compgen -W "${commands} ${modules}" -- "${cur}") )
    fi
}

complete -F _sysext_toggle_completions sysext-toggle
EOF

# Применить в текущей сессии
source /usr/share/bash-completion/completions/sysext-toggle
```

### Для Zsh
```bash
sudo mkdir -p /usr/share/zsh/site-functions
sudo tee /usr/share/zsh/site-functions/_sysext-toggle > /dev/null << 'EOF'
#compdef sysext-toggle

_sysext-toggle() {
    local -a commands modules
    commands=(
        'status:Показать статус всех модулей и OverlayFS'
        'list:Показать список доступных модулей'
        'off-all:Отмонтировать все активные слои'
        'on-all:Смонтировать все активные слои'
    )

    if [[ -d /var/lib/extensions ]]; then
        local f
        for f in /var/lib/extensions/*.raw(N) /var/lib/extensions/*.raw.disabled(N); do
            modules+=( "${${f:t}%.raw(.disabled)#}" )
        done
    fi

    _describe -t commands 'команды' commands
    _describe -t modules 'модули' modules
}

_sysext-toggle "$@"
EOF

# Пересобрать кэш автодополнений в текущей сессии
rm -f ~/.zcompdump* && compinit
```

---

## 7. Важные нюансы и ограничения

* **Конфликты с пакетным менеджером:** Ни `pacman`, ни `dpkg`/`apt` не знают о файлах, смонтированных через `systemd-sysext`. Если вы попытаетесь установить или обновить пакет, файлы которого уже присутствуют в активном `.raw`-модуле, пакетный менеджер выдаст ошибку конфликта файлов (или попытается перезаписать `read-only` слой). Перед крупными обновлениями системы отключайте слои командой `sysext-toggle off-all`.
* **Отсутствие конфигурации в `/etc`:** `systemd-sysext` монтирует только `/usr` и `/opt`. Файлы из `/etc` (например, системные конфиги по умолчанию или правила D-Bus/Polkit) при сборке исключаются. Если программе критически необходимы файлы в `/etc`, для них потребуется отдельный образ `systemd-confext` (`/var/lib/confexts/`).
* **Совместимость библиотек:** В Arch Linux из-за частых обновлений системных библиотек (`glibc`, `gtk`, `qt`) старые `.raw`-образы со временем могут перестать запускаться и потребуют пересборки. В Debian в рамках одного стабильного релиза (например, Debian 12) ABI библиотек заморожен, поэтому собранные образы остаются работоспособными на протяжении всего жизненного цикла релиза.
