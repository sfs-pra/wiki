# Herdr + Pi + pi-extensible-workflows: GTK3/Vala с нуля

## Оглавление

- [Для кого и результат](#для-кого-и-результат)
- [Архитектура](#архитектура)
- [Доверие и безопасность](#доверие-и-безопасность)
- [Требования](#требования)
- [Установка и первый запуск Pi](#установка-и-первый-запуск-pi)
- [Установка Herdr и Pi-интеграции](#установка-herdr-и-pi-интеграции)
- [Установка workflow-пакетов](#установка-workflow-пакетов)
- [Основы настройки Pi](#основы-настройки-pi)
- [Базовый проект и запуск сессии](#базовый-проект-и-запуск-сессии)
- [Простой пример: Hello World](#простой-пример-hello-world)
- [Продвинутый пример: Greeter с оркестрацией](#продвинутый-пример-greeter-с-оркестрацией)
- [Ручное тестирование и обратная связь](#ручное-тестирование-и-обратная-связь)
- [Каталог команд](#каталог-команд)
- [Аналогия с OpenCode](#аналогия-с-opencode)
- [Устранение неполадок](#устранение-неполадок)
- [Обновление и очистка](#обновление-и-очистка)
- [Источники](#источники)

## Для кого и результат

Руководство предназначено для Linux-разработчиков, которые хотят создавать и
проверять GTK3/Vala-проекты с Pi, workflows и Herdr. Результат — Git-проект с
GTK3/Vala Hello World! и понятный путь к Greeter с Meson и тестами.

## Архитектура

- **Herdr** — runtime и наблюдатель терминальных и агентских сессий.
- **Pi** — coding agent, с которым работают в терминале.
- **pi-extensible-workflows** — детерминированный оркестратор workflows.
- **`@piewf/herdr`** — связывает workflow-agent sessions с Herdr.

Pi следует минималистичной модели: базовый coding agent предоставляет `read`,
`write`, `edit` и `bash`, а workflows и subagents добавляются доверенными
расширениями и пакетами.

## Доверие и безопасность

Pi-пакеты и расширения выполняются с правами вашего пользователя к файловой
системе и процессам. Устанавливайте только доверенный код и проверяйте источник
проекта и пакета. Не помещайте API-ключи, токены и другие секреты в репозиторий.
Проверки разрешений могут предоставляться расширением, но не являются
автоматической гарантией изоляции.

## Требования

Нужны Linux с графической пользовательской сессией, Git, Node.js **>= 22.19**,
Vala/GTK3 и учётные данные AI-провайдера. Для GUI процессу нужен доступ к
дисплею: в X11 обычно корректный `DISPLAY`; в Wayland — переменные и доступ,
унаследованные от графического терминала. SSH, `sudo` или service вне GUI-сессии
могут не открыть окно.

### Системные пакеты

Ubuntu/Debian:

```bash
sudo apt update && sudo apt install -y git curl build-essential pkg-config \
  valac libgtk-3-dev meson ninja-build
```

Fedora:

```bash
sudo dnf install git curl gcc pkgconf-pkg-config vala gtk3-devel meson \
  ninja-build
```

Arch Linux:

```bash
sudo pacman -S --needed git curl base-devel pkgconf vala gtk3 meson ninja
```

Установите Node.js способом из официальной документации Node.js. Не используйте
непроверенную команду из случайного источника и убедитесь, что версия не ниже
22.19.

```bash
node --version
valac --version
pkg-config --modversion gtk+-3.0
meson --version
```

## Установка и первый запуск Pi

Установите текущий официальный пакет и проверьте его:

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
pi --version
```

Первый запуск должен создать `~/.pi/agent`. В обычном терминале выполните `pi`,
внутри Pi настройте `/login`, при необходимости `/model` и `/thinking`, затем
выйдите через `/quit`. Впоследствии ввод `/` показывает фактически доступные
команды вашей версии Pi.

## Установка Herdr и Pi-интеграции

На Linux/macOS:

```bash
curl -fsSL https://herdr.dev/install.sh | sh
herdr --version
```

Запускайте Herdr только **снаружи** Herdr. `HERDR_ENV=1` означает вложенный
запуск: выйдите в обычный терминал. Для detach нажмите Ctrl+B, затем Q; для
reattach в обычном терминале выполните `herdr`. Остановить сервер можно так:

```bash
herdr server stop
```

После первичного запуска Pi установите интеграцию:

```bash
herdr integration install pi
herdr integration status
```

Она записывает `~/.pi/agent/extensions/herdr-agent-state.ts`, либо
эквивалентный путь при заданном `PI_CODING_AGENT_DIR`.

### Размер панелей Herdr

Границу split можно перетаскивать мышью. Ctrl+B, затем R включает resize:
H/L меняют ширину, J/K — высоту; стрелки также работают, а Esc или Enter
завершает режим. Изменяется только split ratio, поэтому при одной панели,
отсутствующей подходящей границе, минимальном размере или zoom изменения могут
быть не видны. Ctrl+B, затем Z переключает zoom; Ctrl+B, затем V создаёт split
справа, Ctrl+B, затем минус — split снизу. Размер всего таба определяется
локальным окном терминала, в том числе размером SSH PTY.

Для диагностики внутри Herdr:

```bash
herdr pane layout --current
herdr pane resize --direction right --amount 0.1 --current
```

## Установка workflow-пакетов

Следующие команды являются командами CLI в **обычной оболочке**, а не текстом
активного разговора Pi. Выйдите из работающего Pi либо перезапустите его после
установки:

```bash
pi install npm:pi-extensible-workflows
pi install npm:@piewf/herdr
```

Доверяйте только проверенному источнику проекта и пакета. `@piewf/herdr`
активен только внутри Herdr. Slash-команды ниже, напротив, вводятся в Pi.

Опциональный полностью инспектируемый режим задаётся глобально, не в
репозитории, в `~/.pi/agent/pi-extensible-workflows/settings.json`:

```json
{
  "extensionSettings": {
    "herdr": {
      "enableFullyInspectableMode": true
    }
  }
}
```

## Основы настройки Pi

| Сущность | Назначение |
| --- | --- |
| Prompt template | Markdown, разворачиваемый в slash-команду. |
| Skill | Инструкция/capability по требованию: `/skill:name`. |
| Extension | TypeScript host-код: tools, UI, events, права пользователя. |
| Theme | Только внешний вид TUI. |
| Package | Набор extensions, skills, prompts и themes. |

Глобальные templates находятся в `~/.pi/agent/prompts/*.md`, а templates
доверенного проекта — в `.pi/prompts/*.md`. Extension — доверенный код с
правами пользователя. Package устанавливается через `pi install`.

Выбор модели выполняется через `/model` или Ctrl+L; Ctrl+S в selector сохраняет
выбор как default. `/thinking` или Shift+Tab меняет reasoning, а `/login` нужен,
если provider ещё не настроен. Новый bundled workflow обычно разрешает aliases
developer/reviewer из модели запустившего Pi, если settings не переопределяют
aliases. Уже работающий workflow использует snapshot на момент запуска.

Пример project-local template `.pi/prompts/gtk-review.md`:

```text
---
description: Проверить GTK3/Vala-изменения
argument-hint: "[область проверки]"
---
Проверь GTK3/Vala-изменения в текущем проекте.
Особое внимание удели:
${@:-общей корректности, тестируемой не-GUI логике и Meson-тестам}.
```

После доверия проекту Pi загружает его context. После изменения template или
`AGENTS.md` выполните `/reload` либо перезапустите Pi. Вызовы шаблона:

```text
/gtk-review
/gtk-review обработка пустого ввода
```

`/review-loop` использует тот же механизм Pi prompt template, но поставляется
пакетом и вызывает `workflow`/`reviewLoop`.

## Базовый проект и запуск сессии

Это руководство не обязано находиться в демонстрационном проекте. Создайте
отдельный каталог, Git-baseline и первый commit: workflows/worktrees получат
точку отсчёта.

```bash
mkdir -p ~/src/gtk3-vala-herdr-demo
cd ~/src/gtk3-vala-herdr-demo
git init
printf '# GTK3 Vala demo\n' > README.md
printf 'build/\nhello-world\ngtk3-greeter\n' > .gitignore
cat > AGENTS.md <<'EOF'
# Project instructions

- Use GTK3, not GTK4.
- Keep non-GUI logic testable.
- Run the documented build and test commands.
EOF
git add README.md .gitignore AGENTS.md
git commit -m "chore: initialize project"
```

Из каталога проекта запустите Herdr:

```bash
herdr
```

В его pane запустите Pi:

```bash
pi
```

## Простой пример: Hello World

Внутри Pi отправьте точный запрос. Последний аргумент — ограничение итераций:

```text
/review-loop Создай в текущем Git-проекте минимальное GTK3-приложение
на Vala Hello World!. Добавь main.vala с Gtk.Application,
Gtk.ApplicationWindow и Label «Hello World!», обнови README.md командами
сборки и запуска, не добавляй лишних зависимостей. Проверь код компиляцией
valac --pkg gtk+-3.0 -o hello-world main.vala. 3
```

Ожидаемый `main.vala`:

```vala
using Gtk;

public class HelloWorld : Gtk.Application {
    public HelloWorld () {
        Object (application_id: "org.example.HelloWorld");
    }

    protected override void activate () {
        var window = new Gtk.ApplicationWindow (this);
        window.title = "Hello World!";
        window.default_width = 320;
        window.default_height = 120;
        window.add (new Gtk.Label ("Hello World!"));
        window.show_all ();
    }

    public static int main (string[] args) {
        return new HelloWorld ().run (args);
    }
}
```

Ручная сборка и запуск:

```bash
valac --pkg gtk+-3.0 -o hello-world main.vala
./hello-world
```

Ожидается GTK-окно с текстом `Hello World!`. Это ожидаемый результат, а не
заявление, что GUI был запущен данным руководством.

## Продвинутый пример: Greeter с оркестрацией

Цель: GTK3 Greeter с `Gtk.Entry`, кнопкой, greeting label, обработкой пустого
ввода, Meson, отделённой тестируемой логикой приветствия, тестами и README.

1. Исследуйте репозиторий:

   ```text
   /parallel-scout Изучи текущий репозиторий и предложи минимальную
   структуру GTK3/Vala Greeter с Meson, отделённой функцией формирования
   приветствия и тестами. Не изменяй файлы.
   ```

1. Сравните варианты:

   ```text
   /council Сравни варианты реализации GTK3/Vala Greeter: где разместить
   чистую логику приветствия, как подключить её к Meson и как написать
   тесты без GUI. Дай краткую рекомендацию для этого репозитория.
   ```

   `/council` даёт мнения 2–3 advisors в workflow, но сам по себе не создаёт
   human checkpoint: прочитайте вывод и решите, продолжать ли.

1. Реализуйте с review loop:

   ```text
   /review-loop Реализуй в текущем репозитории GTK3/Vala Greeter:
   Gtk.Entry, кнопку и метку приветствия; пустой ввод должен показывать
   понятное сообщение; вынеси логику приветствия в тестируемый модуль;
   добавь Meson-сборку, автоматические тесты и README с командами;
   итоговый бинарный файл должен называться gtk3-greeter. Сохрани
   существующие полезные файлы и выполни доступные проверки. 4
   ```

1. Проверьте diff:

   ```text
   /parallel-review Проверь текущий uncommitted diff Greeter на
   корректность Vala/GTK3, Meson, обработку пустого ввода, тестируемость
   логики, тесты и ясность README. Сообщи только конкретные проблемы и
   исправления.
   ```

1. Откройте navigator и browser UI траектории:

   ```text
   /workflow
   /workflow trajectory
   ```

### Удалённая Trajectory через SSH

Если Pi на удалённой машине вывел следующее, это URL Trajectory, а не адрес SSH:

```text
Trajectory opened at http://127.0.0.1:8190/
```

На **локальной** машине создайте туннель. Порт 8190 — порт Trajectory, а не
обязательно порт SSH-сервера:

```bash
ssh -N \
  -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:8190:127.0.0.1:8190 \
  user@remote-host
```

Откройте локально <http://127.0.0.1:8190/>. Если SSH-сервер использует иной
порт, добавьте к команде параметр `-p <ssh-port>`. Если Pi вывел другой порт,
замените 8190 согласованно в обеих сторонах forwarding и URL.

Trajectory-server привязан только к remote loopback (`127.0.0.1`); не
выставляйте его на `0.0.0.0`. У Trajectory нет собственной аутентификации или
TLS: аутентификацию и шифрование обеспечивает SSH.

После оркестрации выполните ручную shell-проверку. `meson setup build` нужен
только при первом создании build-каталога. Повторяйте setup с `--wipe` лишь
когда намеренно требуется перенастройка, не по умолчанию.

```bash
meson setup build
meson compile -C build
meson test -C build
./build/gtk3-greeter
```

```text
Вы → Pi (основной агент)
          ├─ parallel-scout → scouts исследуют репозиторий
          ├─ council → advisors предлагают варианты
          ├─ review-loop → implementer ↔ reviewer, до лимита
          └─ parallel-review → reviewers проверяют diff
                         ↓
                  Herdr panes: agent sessions и их состояние
```

Scouts собирают сведения, advisors оценивают варианты, implementer меняет код,
reviewers ищут проблемы. Панели Herdr делают сессии агентов видимыми; если
доступен handoff, активную работу можно передать через Herdr.

## Ручное тестирование и обратная связь

Внутри Pi префикс `!` запускает shell-команду, поэтому используйте:

```text
!meson test -C build
```

В обычной оболочке та же проверка вводится без `!`:

```bash
meson test -C build
```

Проверьте обычное имя, пробелы и пустой ввод. Скриншот можно вставить в
сообщение Pi, если клиент поддерживает вставку изображения.

Шаблон отчёта об ошибке:

```text
Ошибка: [краткое название]
Шаги воспроизведения:
1. ...
2. ...
Фактический результат: ...
Ожидаемый результат: ...
Окружение: дистрибутив, версии GTK/Vala, DISPLAY или Wayland.
Логи: ...
Скриншот: вставлен или приложен.
```

Для небольшой правки дайте прямой запрос, например: «Исправь только текст
сообщения при пустом вводе, обнови тест и покажи diff». Для исправления с
проверкой запустите новый `/review-loop` с отчётом об ошибке. Завершённый
workflow нельзя дополнить: начните новый run. Для активного workflow используйте
`/workflow` и live handoff Herdr, если действие доступно; открытие завершённой
сессии — только inspection. Для самостоятельных агентов: `/subagents` →
**Steer**.

## Каталог команд

### Операции Pi

Настройка `/model`, `/login` и `/thinking` описана в «Основы настройки Pi».

| Команда | Действие |
| --- | --- |
| `/name` | Задать имя текущей сессии. |
| `/session` | Показать файл, ID, сообщения, токены и стоимость сессии. |
| `/resume` | Продолжить предыдущую сессию. |
| `/new` | Создать новую сессию. |
| `/tree` | Показать ветвление в той же сессии. |
| `/fork` | Создать новую сессию от раннего user message. |
| `/clone` | Дублировать активную ветку в новую сессию. |
| `/compact` | Сжать контекст в summary. |
| `/reload` | Перезагрузить project-local настройки и extensions. |
| `/export` | Экспортировать сессию в HTML. |
| `/share` | Создать private GitHub gist. |
| `/quit` | Выйти из Pi. |

Введите `/`, чтобы увидеть фактический список команд установленной версии.

### Prompt-команды пакета

- `/scout` — один foreground scout.
- `/parallel-scout` — условно один scout или workflow с 2–3 scouts.
- `/oracle` — один foreground oracle.
- `/council` — workflow с 2–3 advisors.
- `/review` — один reviewer, по умолчанию проверяет uncommitted diff.
- `/parallel-review` — workflow с 2–3 reviewers.
- `/review-loop` — background workflow с review loop по умолчанию.
- `/deep-research` — workflow: 2–4 researchers и synthesis.
- `/workflow` — navigator workflows.
- `/subagents` — navigator самостоятельных агентов.
- `/workflow trajectory` — browser UI trajectory.

`@piewf/herdr` добавляет contextual actions, но не slash-команду.

## Аналогия с OpenCode

| OpenCode | Pi workflows |
| --- | --- |
| Primary agent | Основной агент Pi |
| Task/subagents | Pi roles/agents |
| Parallel tasks | `parallel` workflows |
| Reviewer loop | `reviewLoop` |
| Child task IDs | run IDs и agent/role IDs |

Оркестрация OpenCode обычно динамическая, тогда как Pi workflows кодируют
детерминированный JavaScript с journal, recovery, checkpoints и budgets. Есть и
прямая интеграция OpenCode: `herdr integration install opencode`; это
руководство остаётся о Pi.

## Устранение неполадок

- **Команда не найдена:** откройте новый терминал, проверьте `PATH` и версию
  после правильной установки.
- **Node старый:** обновите его по официальной документации до >=22.19 и
  проверьте `node --version`.
- **Нет package-команды:** перезапустите Pi, проверьте доверие к источнику,
  выполните `pi list`, затем `/reload`.
- **Нет GTK pkg-config:** установите development-пакет GTK3 и повторите
  `pkg-config --modversion gtk+-3.0`.
- **Нет GUI:** запускайте из графического терминала; проверьте `DISPLAY` или
  наследованную Wayland-сессию.
- **Herdr не определяет Pi:** выполните `herdr agent list`,
  `herdr agent explain --json`, `herdr integration status`; Pi должен быть
  внутри Herdr, без вложенного запуска с `HERDR_ENV=1`.
- **Некорректные settings:** JSON должен быть глобально в
  `~/.pi/agent/pi-extensible-workflows/settings.json` под
  `extensionSettings.herdr.enableFullyInspectableMode`, не в проекте.
- **Inspection/recovery workflow:** откройте `/workflow` и сведения выбранного
  run. Не используйте выдуманное `/workflow resume`: `/workflow` — picker;
  документированная специальная форма — `/workflow trajectory`.

### Редактор показывает NORMAL

`NORMAL` означает Vim-style editor extension или custom editor, а не статус
workflow и не гарантированное поведение stock Pi. Нажмите i для режима INSERT,
введите slash-команду и нажмите Enter; Esc возвращает NORMAL. Если такой editor
не нужен, выполните `pi config` в обычной оболочке, найдите и отключите
соответствующее расширение, затем перезапустите Pi.

## Обновление и очистка

```bash
pi update --all
herdr update
```

`herdr update` относится к прямой script-установке; при установке пакетным
менеджером обновляйте Herdr им. Для демонстрационного проекта удалите только
binary/build и остановите Herdr:

```bash
rm -rf build hello-world gtk3-greeter
herdr server stop
```

Не удаляйте файлы учётных данных ради очистки: управляйте ими безопасным
механизмом провайдера.

## Источники

- Pi: <https://pi.dev/docs/latest>
- Pi prompt templates: <https://pi.dev/docs/latest/prompt-templates>
- Pi sessions: <https://pi.dev/docs/latest/sessions>
- Pi skills: <https://pi.dev/docs/latest/skills>
- Pi themes: <https://pi.dev/docs/latest/themes>
- Pi extensions: <https://pi.dev/docs/latest/extensions>
- Pi packages: <https://pi.dev/docs/latest/packages>
- Pi settings: <https://pi.dev/docs/latest/settings>
- Node.js: <https://nodejs.org/en/download>
- Herdr: <https://herdr.dev/docs/>
- Быстрый старт Herdr: <https://herdr.dev/docs/quick-start/>
- Клавиатура Herdr: <https://herdr.dev/docs/keyboard/>
- Удалённая работа Herdr: <https://herdr.dev/docs/how-to-work/>
- Интеграции Herdr: <https://herdr.dev/docs/integrations/>
- pi-extensible-workflows: <https://vekexasia.github.io/pi-extensible-workflows/>
- Herdr для workflows: <https://vekexasia.github.io/pi-extensible-workflows/herdr.html>
- Subagents: <https://vekexasia.github.io/pi-extensible-workflows/subagents.html>
- Установка Vala: <https://docs.vala.dev/get-started/installation-guide>
- GTK+ Vala bindings: <https://www.gtk.org/docs/language-bindings/vala/>
- Архивный GTK3 Vala sample GNOME: <https://wiki.gnome.org/Projects/Vala/GTKSample>
- Видео пользователя (дополнительный обзор):
  <https://www.youtube.com/watch?v=qAiivspEHmU>
- Alejandro AO, «Pi Agent – Crash Course | Minimal Coding Agent»
  (дополнительный обзор): <https://www.youtube.com/watch?v=N30XGyPrr6I>
  Тема по metadata автора: 0:51 — Pi; 3:46 — models; 5:28 — custom prompts;
  8:05 — skills; 11:26 — themes; 13:17 — context files; 15:13 — extensions;
  20:32 — packages; 22:02 — sessions.

Официальная документация — источник истины; видео служит дополнением.
