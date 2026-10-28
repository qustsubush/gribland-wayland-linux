# gribland-wayland-linux
Скрипт запуска Gribland launcher на Arch Linux (Wayland).

- запускает bootstrap через `jdk21-openjdk`;
- запускает GTK через XWayland (`GDK_BACKEND=x11`), иначе JavaFX 8 падает на Wayland с `SIGSEGV`;
- выставляет бит исполнения на скачанные лаунчером JRE (иначе `error=13, Permission denied`);
- переносит bootstrap из `~/Downloads` в постоянную папку;
- скачивает иконку для ярлыка.

> Скрипт сделан под Arch и использует `pacman`. На других дистрибутивах автоустановка Java не сработает.

## Требования

| Что | Зачем | Пакет |
|---|---|---|
| Java 21 | запуск bootstrap | `jdk21-openjdk` (скрипт предложит поставить сам) |
| XWayland | окно JavaFX 8 | `xorg-xwayland` (должен быть непустой `$DISPLAY`) |
| curl | скачивание иконки | `curl` |

## Установка

1. Скачай bootstrap (`gribland-bootstrap-*.jar`) с <https://gribland.net/> в `~/Downloads`.
2. Установи и запусти скрипт:

   ```bash
    # Установка
   mkdir -p gribland-wayland-linux
   curl -fSL -o gribland-wayland-linux/gribland https://github.com/qustsubush/gribland-wayland-linux/releases/latest/download/gribland-wayland-linux
   chmod +x gribland-wayland-linux/gribland
    # Запуск
   ./gribland-wayland-linux/gribland
   ```
   
При первом запуске скрипт:

1. скачает иконку в `~/.local/share/icons/gribland.png`;
2. перенесёт jar из `~/Downloads` в `~/.local/share/gribland-bootstrap/`;
3. спросит, ставить ли `jdk21-openjdk`, если его нет (Enter = да, `n` = отмена);
4. запустит bootstrap, а он сам скачает лаунчер и JRE в `~/gribland`.

> Первый запуск лучше делать из терминала: установка Java требует пароль `sudo`.

## Ярлык в меню приложений

Скопируй команду ниже. Она сама создаст `.desktop`-файл. Команда подставит твой `$HOME` в пути сама:

```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/gribland.desktop <<EOF
[Desktop Entry]
Type=Application
Name=Gribland
Comment=Gribland launcher
Exec=$HOME/gribland-wayland-linux/gribland
Icon=$HOME/.local/share/icons/gribland.png
Terminal=false
Categories=Game;
EOF
update-desktop-database ~/.local/share/applications 2>/dev/null
```

После этого Gribland появится в меню приложений.

Важно:

- Пути в `Exec=` и `Icon=` должны быть абсолютными: `~` и `$HOME` в `.desktop`-файлах не раскрываются.
- Иконка создаётся при первом запуске скрипта. Если её ещё нет, сначала запусти `gribland` один раз.
- `Terminal=false` подходит, когда Java уже установлена. Если запускаешь ярлыком впервые и Java нет, используй `Terminal=true` или поставь её заранее:

  ```bash
  sudo pacman -S jdk21-openjdk
  ```

- Иконка берётся с `og_image.png`, это картинка для превью ссылок. Она может быть прямоугольной и выглядеть в меню сжатой. Если так, подставь свою квадратную PNG в `~/.local/share/icons/gribland.png`.

## Где что лежит

| Путь | Что там |
|---|---|
| `~/gribland-wayland-linux/gribland` | скрипт запуска |
| `~/gribland-wayland-linux/gribland-bootstrap/` | jar bootstrap |
| `~/.local/share/icons/gribland.png` | иконка |
| `~/.local/share/applications/gribland.desktop` | ярлык |
| `~/gribland/` | данные лаунчера (создаёт bootstrap): `bin/bootstrap`, `bin/runtime` |

## Решение проблем

**Скрипт пишет «Bootstrap не найден».**
В `~/Downloads` и в `~/.local/share/gribland-bootstrap/` нет файла `gribland-bootstrap-*.jar`. Скрипт ищет только по этой маске. Если файл называется иначе, переименуй его.

**`DISPLAY пуст: XWayland не запущен`.**
Установи `xorg-xwayland` и убедись, что композитор его запускает. Проверка: `echo $DISPLAY`.

**Лаунчер не открывается, в консоли `SIGSEGV` и `GDK_IS_X11_DISPLAY`.**
GTK работает в режиме Wayland. Скрипт уже задаёт `GDK_BACKEND=x11`. Если запускаешь bootstrap вручную, добавь эту переменную сам.

**`error=13, Permission denied` на `.../runtime/jre25/.../bin/java`.**
Лаунчер распаковал JRE без прав на исполнение. Скрипт чинит это в фоне каждые 2 секунды. Вручную:

```bash
find ~/gribland/bin/runtime -type f \( -path '*/bin/*' -o -name jspawnhelper \) -exec chmod +x {} +
```

Если ошибка осталась, проверь, что домашняя папка не смонтирована с `noexec`:

```bash
findmnt -T ~/gribland -o TARGET,OPTIONS
```

**Окно рисуется с артефактами или не появляется без ошибок.**
Попробуй программный рендеринг JavaFX. В последней строке скрипта запуск идёт командой `"$JAVA_BIN" -jar "$JAR" "$@"`, а переменная `JAVA_TOOL_OPTIONS` передаётся и дочерним процессам:

```bash
JAVA_TOOL_OPTIONS="-Dprism.order=sw" gribland
```

**Нужен лог краша.**
Рядом с запуском JRE создаются `hs_err_pid*.log` и `core*`. Их можно удалять:

```bash
rm -f ~/gribland/bin/bootstrap/core* ~/gribland/bin/bootstrap/hs_err_pid*.log
```

## Что делает скрипт

1. Создаёт папки для jar и иконки.
2. Если иконки нет, скачивает её через `curl` (при ошибке продолжает без неё).
3. Если в `~/Downloads` лежит `gribland-bootstrap-*.jar`, переносит самый новый в `~/gribland-wayland-linux/gribland-bootstrap/` и **удаляет все остальные `gribland-bootstrap-*.jar` из `~/Downloads`**.
4. Проверяет `jdk21-openjdk`, при отсутствии спрашивает об установке через `pacman`.
5. Проверяет `$DISPLAY`.
6. В фоне выставляет `+x` на файлы в `~/gribland/bin/runtime`.
7. Запускает bootstrap с `GDK_BACKEND=x11`.

## Удаление

```bash
# Скрипт запуска и перенесённый bootstrap (jar)
rm -rf ~/gribland-wayland-linux

# Данные лаунчера, скачанные bootstrap (bin/bootstrap, bin/runtime)
rm -rf ~/gribland

# Иконка
rm -f ~/.local/share/icons/gribland.png

# Ярлык в меню приложений и на рабочем столе
rm -f ~/.local/share/applications/gribland.desktop
update-desktop-database ~/.local/share/applications 2>/dev/null

# Java (если больше не нужна)
sudo pacman -Rns jdk21-openjdk
```

## Известные ограничения

- Каждая проблема, которую решает скрипт (X11-бэкенд, права на JRE), найдена при отладке на одном пк (Arch, Wayland). Сам скрипт целиком в таком виде не прогонялся. На других конфигурациях поведение может отличаться.
- Поддерживается только `pacman`.
