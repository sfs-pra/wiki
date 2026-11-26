# `systemd-sysext` и `systemd-confext` в Arch Linux и Debian: руководство с проверкой на практике

Что проверено, а что взято из документации, отмечено явно. Проверки: systemd 262, Arch Linux
(сборка puZZle 2611), виртуальная машина QEMU; установленная система на ext4 и live-система с
корнем на aufs.

## Содержание

- [1. Как это работает](#1-как-это-работает)
- [2. Совместимость: файл `extension-release`](#2-совместимость-файл-extension-release)
- [3. Только чтение и режим `--mutable`](#3-только-чтение-и-режим---mutable)
- [4. Ручная сборка и подключение](#4-ручная-сборка-и-подключение)
- [5. `/etc` и `/var`: confext и копирование](#5-etc-и-var-confext-и-копирование)
- [6. Кэши, службы, пользователи](#6-кэши-службы-пользователи)
- [7. Упаковка GUI-приложений (на примере `galculator`)](#7-упаковка-gui-приложений-на-примере-galculator)
- [8. Скрипт-переключатель `sysext-toggle`](#8-скрипт-переключатель-sysext-toggle)
- [9. Подключение при загрузке](#9-подключение-при-загрузке)
- [10. Модуль внутри пакета pacman (`.pkg.tar`)](#10-модуль-внутри-пакета-pacman-pkgtar)
- [11. Live-системы (frugal, aufs/overlayfs)](#11-live-системы-frugal-aufsoverlayfs)
- [12. Автодополнение (`Tab`)](#12-автодополнение-tab)
- [13. Ограничения](#13-ограничения)

---

## 1. Как это работает

**`systemd-sysext`** накладывает образы-расширения поверх `/usr` и `/opt` через OverlayFS.
**`systemd-confext`** делает то же самое для `/etc`.

- **Форматы образов:** каталог, образ GPT (по желанию с dm-verity), «голая» файловая система
  SquashFS, EROFS или ext4 в файле `*.raw`.
- **Где ищутся:** sysext — `/etc/extensions/`, `/run/extensions/`, `/var/lib/extensions/`;
  confext — `/run/confexts/`, `/var/lib/confexts/`, `/usr/lib/confexts/`,
  `/usr/local/lib/confexts/`. Каталог в этих местах — тоже расширение, поэтому туда можно
  положить символическую ссылку на смонтированный каталог (см. разделы 10 и 11).
- **Что попадает в систему:** sysext — только `/usr` и `/opt`; confext — только `/etc`. Файлы
  образа в `/var` и других местах игнорируются (проверено: записи pacman из
  `/var/lib/pacman/local` образа не видны).
- **`refresh`** снимает overlay и собирает его заново **со всеми** расширениями сразу; на короткое
  время пропадают файлы всех расширений, а не только изменённого.
- **Включать/выключать отдельный образ** нечем: подключается всё, что лежит в каталогах поиска.
  Отключить — убрать образ (или пустой каталог с тем же именем в `/etc/extensions/` для маскировки).

Требования к образу:

1. корень образа соответствует корню системы: файлы в `usr/...`, `opt/...` (для confext — `etc/...`);
2. файл `usr/lib/extension-release.d/extension-release.ИМЯ` (для confext —
   `etc/extension-release.d/extension-release.ИМЯ`), где `ИМЯ` совпадает с именем образа без
   `.raw`;
3. образ и каталоги принадлежат root и не доступны на запись другим.

---

## 2. Совместимость: файл `extension-release`

Перед подключением `ID=` из `extension-release` сравнивается с `/etc/os-release`. По документации:
если `ID` не `_any`, должен совпасть `SYSEXT_LEVEL=` (если задан), иначе `VERSION_ID=`. Для
confext — `CONFEXT_LEVEL=`/`VERSION_ID=`.

- **Debian** — фиксированные релизы, в `os-release` есть `VERSION_ID`:
  ```ini
  ID=debian
  VERSION_ID=12
  ```
- **Arch Linux** — rolling, `VERSION_ID` нет. Вариант из документации — уровень совместимости:
  ```ini
  ID=arch
  SYSEXT_LEVEL=1
  ```
  Добавив `SYSEXT_LEVEL=2` в `/etc/os-release` хоста, можно отсечь старые образы. *Этот вариант на
  Arch без `VERSION_ID` мы не проверяли.*
- **Без проверки** (проверено):
  ```ini
  ID=_any
  ```
  Подходит для статических программ и для образов, собранных под ту же систему, — ответственность за
  совместимость библиотек на вас.

Полезные поля (из документации; `EXTENSION_RELOAD_MANAGER` проверен):

```ini
EXTENSION_RELOAD_MANAGER=1                     # daemon-reload после merge/refresh/unmerge
EXTENSION_RESTART_UNITS="my.service"           # перезапустить юниты (подразумевает reload)
EXTENSION_RELOAD_OR_RESTART_UNITS="other.service"
ARCHITECTURE=x86-64
```

---

## 3. Только чтение и режим `--mutable`

**По умолчанию, пока подключено хотя бы одно расширение, `/usr` и `/opt` (а с confext — `/etc`)
доступны только для чтения — даже на обычной системе с записываемым корнем.** Проверено:

```
touch /usr/share/test           → Read-only file system
gtk-update-icon-cache …         → Failed to open file …: Read-only file system
update-desktop-database …       → could not be updated
```

То есть в режиме по умолчанию не работают ни установка пакетов, ни обновление кэшей после
подключения (в т. ч. то, что предлагала прежняя версия этой статьи).

С systemd 256 есть `--mutable=` (`Mutable=` в `sysext.conf`):

| Режим | Что происходит |
|---|---|
| `no` (по умолчанию) | только чтение |
| `auto` | запись разрешена, если есть каталог/ссылка в `/var/lib/extensions.mutable/` |
| `yes` | то же, каталоги создаются сами |
| `ephemeral` | запись во временный каталог, пропадает после `unmerge` |
| `import`, `ephemeral-import` | варианты с подмешиванием `/var/lib/extensions.mutable/` |

Чтобы запись шла **в настоящие каталоги**, как без расширений, нужны ссылки:

```bash
sudo mkdir -p /var/lib/extensions.mutable
sudo ln -s /usr /var/lib/extensions.mutable/usr
sudo ln -s /opt /var/lib/extensions.mutable/opt
sudo ln -s /etc /var/lib/extensions.mutable/etc   # для confext
sudo systemd-sysext refresh --mutable=auto
```

Проверено на ext4: overlay получает `upperdir=/usr`; запись в `/usr`, `gtk-update-icon-cache`,
`update-desktop-database`, `ldconfig`, `pacman -U` работают, записанное остаётся после
`unmerge`. Файлы самих расширений в `/usr` не копируются.

На корне aufs или overlayfs (live-системы) такой режим **не работает**: ядро не принимает их как
слой записи (`overlayfs: upper fs missing required features`, `Failed to merge hierarchies`).
`ephemeral` там работает, но всё записанное пропадает при отключении.

---

## 4. Ручная сборка и подключение

```bash
mkdir -p mytools/usr/bin mytools/usr/lib/extension-release.d
# бинарники — в mytools/usr/bin/
printf 'ID=arch\nSYSEXT_LEVEL=1\n' > mytools/usr/lib/extension-release.d/extension-release.mytools
mksquashfs mytools mytools.raw -noappend -comp zstd -all-root   # -all-root: владелец root
```

Подключение и отключение:

```bash
sudo mkdir -p /var/lib/extensions
sudo mv mytools.raw /var/lib/extensions/
sudo systemctl enable systemd-sysext.service          # подключать при загрузке
sudo systemd-sysext refresh --mutable=auto            # подключить сейчас (см. раздел 3)
systemd-sysext status

sudo mv /var/lib/extensions/mytools.raw /var/lib/extensions/mytools.raw.disabled
sudo systemd-sysext refresh --mutable=auto            # без этого образа
sudo systemd-sysext unmerge                           # снять все
```

Время на установленной системе: `refresh` с двумя расширениями — около 0,02 с; основное время
уходит на кэши (раздел 6).

---

## 5. `/etc` и `/var`: confext и копирование

- **`/etc`** — `systemd-confext`. Тот же образ можно подключить и как sysext, и как confext, если в
  нём есть оба `extension-release` (`usr/lib/extension-release.d/…` и `etc/extension-release.d/…`).
  confext монтирует `/etc` с `noexec`; для скриптов в `/etc` нужен `--noexec=false`. Проверено:
  `systemd-confext refresh --mutable=auto --noexec=false` с `/var/lib/extensions.mutable/etc` →
  `/etc` — файлы модуля видны, запись в `/etc` работает.
- **`/var`** не подключается ничем. Рабочий способ — скопировать при подключении только то, чего
  нет в системе, запомнить список с контрольными суммами и при отключении удалить неизменённое.
  Так в систему попадают, например, записи `/var/lib/pacman/local`, и `pacman -Q` видит пакеты
  модуля (проверено).

---

## 6. Кэши, службы, пользователи

sysext не делает того, что при загрузке делают службы systemd. После `merge`/`refresh`/`unmerge`
(в режиме `--mutable`) нужно самому:

```bash
systemctl daemon-reload               # или EXTENSION_RELOAD_MANAGER=1
systemd-sysusers                      # пользователи из usr/lib/sysusers.d
systemd-tmpfiles --create
udevadm control --reload
busctl call org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus ReloadConfig
ldconfig
# и только то, что затрагивает модуль:
depmod -a "$(uname -r)"                                   # usr/lib/modules
update-desktop-database -q /usr/share/applications        # ярлыки
PKGSYSTEM_ENABLE_FSYNC=0 update-mime-database /usr/share/mime
glib-compile-schemas /usr/share/glib-2.0/schemas
gtk-update-icon-cache -q -t -f /usr/share/icons/hicolor
fc-cache -s
```

- `update-mime-database` без `PKGSYSTEM_ENABLE_FSYNC=0` (так его зовёт хук pacman) на медленном
  диске занял 54 с; с этой переменной подключение двух модулей вместе со всеми шагами — 2,3 с.
- После `unmerge` кэши, собранные при подключённом модуле, устаревают — их тоже нужно обновить.
- `gtk-update-icon-cache` из GTK4 пишет `icon-theme.cache` (без точки).
- Юниты из `*.wants` модуля сами не запускаются: `systemctl start --no-block ЮНИТ` после
  подключения, `systemctl stop` — до отключения.

---

## 7. Упаковка GUI-приложений (на примере `galculator`)

Чтобы приложение появилось в меню без перезапуска сеанса, в образе должны быть
`usr/share/applications/*.desktop` и иконки, а после `refresh` — обновление кэшей (раздел 6),
что возможно только в режиме `--mutable` (раздел 3).

### Автоматическая сборка для Arch Linux (`build-sysext-arch.sh`)

Скачивает пакет и **только недостающие** зависимости, распаковывает, собирает `.raw`. *Скрипт целиком не проверялся.*
Базы пакетов обновите заранее полным `pacman -Syu`: `pacman -Sy` без обновления системы — частичное
обновление, которое Arch не поддерживает.

```bash
#!/usr/bin/env bash
set -euo pipefail

PKG_NAME="${1:-galculator}"
EXT_NAME="${2:-$PKG_NAME}"
WORK_DIR=$(mktemp -d -t sysext-arch-XXXXXX)
trap 'sudo rm -rf "$WORK_DIR"' EXIT

for cmd in pacman mksquashfs tar; do
    command -v "$cmd" &>/dev/null || { echo "Ошибка: нет '$cmd' (squashfs-tools?)"; exit 1; }
done

mkdir -p "$WORK_DIR/pkgs" "$WORK_DIR/rootfs"

echo "==> Скачивание $PKG_NAME и недостающих зависимостей..."
sudo pacman -Sw --cachedir "$WORK_DIR/pkgs" --noconfirm "$PKG_NAME"

echo "==> Распаковка..."
for pkg in "$WORK_DIR"/pkgs/*.pkg.tar*; do
    [ -e "$pkg" ] || continue
    sudo tar -xf "$pkg" -C "$WORK_DIR/rootfs" \
        --exclude='.BUILDINFO' --exclude='.MTREE' --exclude='.PKGINFO' --exclude='.INSTALL'
done

# sysext видит только /usr и /opt; /etc пакета (правила D-Bus, polkit, конфиги) понадобится
# отдельный confext-образ (раздел 5), /var — копирование
echo "==> Удаление всего вне /usr и /opt..."
sudo find "$WORK_DIR/rootfs" -mindepth 1 -maxdepth 1 ! -name usr ! -name opt -exec rm -rf {} +

sudo mkdir -p "$WORK_DIR/rootfs/usr/lib/extension-release.d"
printf 'ID=arch\nSYSEXT_LEVEL=1\n' |
    sudo tee "$WORK_DIR/rootfs/usr/lib/extension-release.d/extension-release.${EXT_NAME}" >/dev/null

sudo mksquashfs "$WORK_DIR/rootfs" "${EXT_NAME}.raw" -noappend -comp zstd -all-root
sudo chown "$USER:$USER" "${EXT_NAME}.raw"
echo "==> Готово: ${EXT_NAME}.raw"
```

### Автоматическая сборка для Debian (`build-sysext-debian.sh`)

`apt-get install --download-only` скачивает пакет с отсутствующими зависимостями, `dpkg-deb -x`
распаковывает; `ID` и `VERSION_ID` берутся из `/etc/os-release`. *Скрипт не проверялся.*

```bash
#!/usr/bin/env bash
set -euo pipefail

PKG_NAME="${1:-galculator}"
EXT_NAME="${2:-$PKG_NAME}"
WORK_DIR=$(mktemp -d -t sysext-deb-XXXXXX)
trap 'sudo rm -rf "$WORK_DIR"' EXIT

for cmd in apt-get dpkg-deb mksquashfs; do
    command -v "$cmd" &>/dev/null || { echo "Ошибка: нет '$cmd'"; exit 1; }
done

mkdir -p "$WORK_DIR/pkgs" "$WORK_DIR/rootfs"
sudo apt-get update -qq
sudo apt-get install --download-only --yes -o Dir::Cache::archives="$WORK_DIR/pkgs" "$PKG_NAME"

for deb in "$WORK_DIR"/pkgs/*.deb; do
    [ -e "$deb" ] || continue
    sudo dpkg-deb -x "$deb" "$WORK_DIR/rootfs"
done

# usrmerge (Debian 12+): старые /bin, /lib — в /usr
for dir in bin sbin lib lib64; do
    if [[ -d "$WORK_DIR/rootfs/$dir" && ! -L "$WORK_DIR/rootfs/$dir" ]]; then
        sudo mkdir -p "$WORK_DIR/rootfs/usr/$dir"
        sudo cp -a "$WORK_DIR/rootfs/$dir/." "$WORK_DIR/rootfs/usr/$dir/"
        sudo rm -rf "$WORK_DIR/rootfs/$dir"
    fi
done

sudo find "$WORK_DIR/rootfs" -mindepth 1 -maxdepth 1 ! -name usr ! -name opt -exec rm -rf {} +

OS_ID=$(. /etc/os-release && echo "${ID:-debian}")
OS_VER=$(. /etc/os-release && echo "${VERSION_ID:-}")
sudo mkdir -p "$WORK_DIR/rootfs/usr/lib/extension-release.d"
{ echo "ID=${OS_ID}"; [[ -n "$OS_VER" ]] && echo "VERSION_ID=${OS_VER}"; } |
    sudo tee "$WORK_DIR/rootfs/usr/lib/extension-release.d/extension-release.${EXT_NAME}" >/dev/null

sudo mksquashfs "$WORK_DIR/rootfs" "${EXT_NAME}.raw" -noappend -comp zstd -all-root
sudo chown "$USER:$USER" "${EXT_NAME}.raw"
echo "==> Готово: ${EXT_NAME}.raw"
```

---

## 8. Скрипт-переключатель `sysext-toggle`

Включает/выключает `.raw`-модуль, подключает в режиме `--mutable` (иначе обновить кэши нельзя) и
обновляет кэши. Сохраните в `/usr/local/bin/sysext-toggle`, `chmod +x`.
*Команды из скрипта проверены по отдельности, сам скрипт целиком — нет.*

```bash
#!/usr/bin/env bash
set -euo pipefail

EXT_DIR=/var/lib/extensions
MUT=/var/lib/extensions.mutable

[[ $EUID -eq 0 ]] || exec sudo "$0" "$@"
mkdir -p "$EXT_DIR" "$MUT"
# запись — в настоящие /usr и /opt (раздел 3)
for d in usr opt; do [[ -e $MUT/$d ]] || ln -s "/$d" "$MUT/$d"; done

refresh() {
    if compgen -G "$EXT_DIR/*.raw" >/dev/null; then
        systemd-sysext refresh --mutable=auto
    else
        systemd-sysext unmerge
    fi
}

update_caches() {
    echo "==> Обновление кэшей..."
    systemctl daemon-reload
    ldconfig
    update-desktop-database -q /usr/share/applications 2>/dev/null || true
    PKGSYSTEM_ENABLE_FSYNC=0 update-mime-database /usr/share/mime 2>/dev/null || true
    gtk-update-icon-cache -q -t -f /usr/share/icons/hicolor 2>/dev/null || true
}

show_status() {
    shopt -s nullglob
    local f found=0
    echo "=== Модули в $EXT_DIR ==="
    for f in "$EXT_DIR"/*.raw "$EXT_DIR"/*.raw.disabled; do
        found=1
        f=$(basename "$f")
        if [[ $f == *.raw ]]; then echo "  [ВКЛ]  ${f%.raw}"; else echo "  [ВЫКЛ] ${f%.raw.disabled}"; fi
    done
    ((found)) || echo "  (модули не найдены)"
    echo
    systemd-sysext status
}

toggle_module() {
    local name=$1 on="$EXT_DIR/$1.raw" off="$EXT_DIR/$1.raw.disabled"
    if [[ -f $on ]]; then
        mv "$on" "$off"; refresh; update_caches; echo "==> '$name' отключён."
    elif [[ -f $off ]]; then
        mv "$off" "$on"; refresh; update_caches; echo "==> '$name' подключён."
    else
        echo "Ошибка: модуль '$name' не найден в $EXT_DIR" >&2; exit 1
    fi
}

case "${1:-status}" in
    status|list) show_status ;;
    off-all) systemd-sysext unmerge; update_caches ;;
    on-all) refresh; update_caches ;;
    *) toggle_module "$1" ;;
esac
```

Примеры:

```bash
sudo cp galculator.raw /var/lib/extensions/
sysext-toggle galculator      # переключить
sysext-toggle                 # список и статус
sysext-toggle off-all         # снять все (например, перед крупным обновлением)
sysext-toggle on-all
```

Скрипт не трогает `/etc` и `/var`, не запускает службы и пользователей модуля — для этого см.
разделы 5 и 6.

---

## 9. Подключение при загрузке

`systemd-sysext.service` (и `systemd-confext.service`) подключает образы из каталогов поиска
**до** `sysinit.target`. Если образы надо сначала смонтировать самому (раздел 10), нужна своя
служба с тем же порядком (порядок проверен на двух перезагрузках):

```ini
[Unit]
Description=Mount my extension images
DefaultDependencies=no
Wants=modprobe@loop.service modprobe@squashfs.service modprobe@overlay.service
After=local-fs.target modprobe@loop.service modprobe@squashfs.service modprobe@overlay.service
Before=sysinit.target systemd-tmpfiles-setup.service
Conflicts=shutdown.target
Before=shutdown.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/my-extensions-mount
# systemd-sysusers.service к этому моменту уже отработал
ExecStartPost=/usr/bin/systemd-sysusers

[Install]
WantedBy=sysinit.target
```

Подводные камни, на которые наткнулись: `Before=systemd-sysusers.service` даёт цикл зависимостей
(`local-fs.target: Found ordering cycle`), а без `modprobe@loop` монтирование падает с «failed to
set up loop device» — модули `loop`, `squashfs`, `overlay` ещё не загружены.

---

## 10. Модуль внутри пакета pacman (`.pkg.tar`)

Образ SquashFS можно положить одним файлом в **несжатый** пакет pacman (`.pkg.tar`): данные элемента
tar лежат непрерывно, и loop-устройство со смещением монтирует модуль прямо из файла пакета, без
распаковки. pacman при этом даёт репозиторий, зависимости и скачивание. Скорость чтения из такого
пакета и из обычного файла `.pfs` на HDD одинакова (265 МБ: 8,75–8,94 с против 8,85–9,01 с).

sysext не открывает `.pkg.tar` сам — его надо смонтировать и подложить каталог:

```bash
P=pfs-rclone-2611.sf01-1-x86_64.pkg.tar
OFF=$(python3 -c 'import sys,tarfile; print(next(m.offset_data for m in tarfile.open(sys.argv[1]) if m.name.endswith(".pfs")))' "$P")
mkdir -p /run/my/rclone /run/my/rel/usr/lib/extension-release.d /run/my/img /run/extensions
mount -t squashfs -o loop,ro,offset="$OFF" "$P" /run/my/rclone
printf 'ID=_any\n' > /run/my/rel/usr/lib/extension-release.d/extension-release.rclone
# файл extension-release добавляется слоем поверх модуля, сам модуль не меняется
mount -t overlay overlay -o lowerdir=/run/my/rel:/run/my/rclone /run/my/img
ln -s /run/my/img /run/extensions/rclone
systemd-sysext refresh --mutable=auto
```

Проверено: `rclone version` работает. Полная реализация (sysext + confext, копия `/var`, кэши,
службы, служба загрузки) — утилита `pfsext` из puZZle: подключение двух модулей — 2,3 с,
отключение — 1,3 с.

---

## 11. Live-системы (frugal, aufs/overlayfs)

На live-системе с корнем на aufs/overlayfs sysext подключается (проверено), но:

- без `--mutable` `/usr` становится только для чтения — ломаются pacman и кэши;
- `--mutable=yes` с записью в `/usr` не подключается: aufs/overlayfs нельзя сделать слоем записи;
- `ephemeral` работает, но записанное пропадает при отключении;
- `/etc` и `/var` из образа не видны, а `refresh` пересобирает overlay целиком.

Объединение корня (aufs/overlayfs) sysext не заменяет — он кладёт второй overlay поверх `/usr`.
Для live-систем удобнее штатное подключение модулей ветками объединения (например, `pfsload` из
pfs-utils), для установленных систем — sysext в режиме `--mutable`.

---

## 12. Автодополнение (`Tab`)

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
            modules=$(find /var/lib/extensions -maxdepth 1 \( -name "*.raw" -o -name "*.raw.disabled" \) -printf "%f\n" 2>/dev/null | sed -E 's/\.raw(\.disabled)?$//')
        fi
        COMPREPLY=( $(compgen -W "${commands} ${modules}" -- "${cur}") )
    fi
}
complete -F _sysext_toggle_completions sysext-toggle
EOF
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
rm -f ~/.zcompdump* && compinit
```

*Автодополнение не проверялось.*

---

## 13. Ограничения

- **Только чтение по умолчанию** — см. раздел 3; `--mutable` нужен systemd 256+.
- **`refresh` пересобирает всё.** Процессы, запущенные при подключённом модуле, держат старый
  overlay: после отключения модуля его файл и loop-устройство остаются занятыми, пока эти
  процессы работают (на практике — до перезагрузки). Файл модуля удалить можно сразу.
  Работающая программа из отключённого модуля продолжает работать (проверено на `rclone rcd`).
- **Пакетный менеджер не знает о файлах расширений.** Без копии `/var/lib/pacman/local`
  (раздел 5) `pacman -Q` их не видит. Перед крупным обновлением системы расширения лучше снять.
- **Порядок наложения** — по именам образов; удаления файлов (whiteout) — только в формате
  overlayfs.
- **Совместимость библиотек.** В Arch из-за частых обновлений `glibc`, `gtk`, `qt` старые образы со
  временем перестают запускаться и требуют пересборки. В Debian в пределах релиза ABI заморожен.
- **Нет зависимостей между образами** — документация прямо говорит, что sysext не система пакетов.
  Зависимости дают обёртки, например модуль внутри пакета pacman (раздел 10).
