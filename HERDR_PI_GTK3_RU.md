# Herdr + Pi + pi-extensible-workflows: GTK3/Vala с нуля

## Для кого и результат

Руководство предназначено для Linux-разработчиков, которые хотят создавать и
проверять GTK3/Vala-проекты с Pi, workflows и Herdr. Результат — Git-проект с
GTK3/Vala Hello World! и понятный путь к Greeter с Meson и тестами.

## Архитектура

- **Herdr** — runtime и наблюдатель терминальных и агентских сессий.
- **Pi** — coding agent, с которым работают в терминале.
- **pi-extensible-workflows** — детерминированный оркестратор workflows.
- **`@piewf/herdr`** — связывает workflow-agent sessions с Herdr.

## Доверие и безопасность

Pi-пакеты и расширения выполняются с правами вашего пользователя к файловой
системе и процессам. Устанавливайте только доверенный код и проверяйте источник
проекта и пакета. Не помещайте API-ключи, токены и другие секреты в репозиторий.

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
git add README.md .gitignore
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
заявление, что GUI был запущен данным руководством. `/review-loop` не встроена
в Pi: это prompt template `pi-extensible-workflows`, вызывающий инструмент
`workflow` и зарегистрированный `reviewLoop`.

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

### Встроенные Pi

Примеры: `/login`, `/model`, `/thinking`, `/new`, `/resume`, `/tree`, `/fork`,
`/clone`, `/compact`, `/reload`, `/export`, `/import`, `/share`, `/quit`.
Фактический список покажет `/`.

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
- Node.js: <https://nodejs.org/en/download>
- Herdr: <https://herdr.dev/docs/>
- Интеграции Herdr: <https://herdr.dev/docs/integrations/>
- pi-extensible-workflows: <https://vekexasia.github.io/pi-extensible-workflows/>
- Herdr для workflows: <https://vekexasia.github.io/pi-extensible-workflows/herdr.html>
- Subagents: <https://vekexasia.github.io/pi-extensible-workflows/subagents.html>
- Установка Vala: <https://docs.vala.dev/get-started/installation-guide>
- GTK+ Vala bindings: <https://www.gtk.org/docs/language-bindings/vala/>
- Архивный GTK3 Vala sample GNOME: <https://wiki.gnome.org/Projects/Vala/GTKSample>
- Видео пользователя (дополнительный обзор):
  <https://www.youtube.com/watch?v=qAiivspEHmU>

Официальная документация — источник истины; видео служит дополнением.
