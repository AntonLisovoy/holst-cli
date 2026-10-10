# Holst CLI

Управление [Holst](https://holst.so) из терминала и AI-агентов: рабочие
пространства, доски, чтение содержимого, заметки, текст, фигуры, стрелки и бэкапы.

## Установка

macOS Apple Silicon и Linux x86-64. Python, Node и Yjs уже включены.

```sh
brew install AntonLisovoy/tap/holst-cli
```

Или:

```sh
curl -LsSf https://raw.githubusercontent.com/AntonLisovoy/holst-cli/main/install.sh | sh
```

Установщик проверяет SHA-256 и добавляет команду в `~/.local/bin`.
При необходимости добавьте эту папку в `PATH`. Для обновления повторите
установку; для Homebrew выполните `brew update && brew upgrade holst-cli`.

## Начало работы

```sh
holst --version
holst auth login --email you@example.com
holst --help
```

Нужен аккаунт Holst. Вход сохраняет cookie сессии и токен.
Персональный API-токен не подходит для управления досками.

## Навык для агента

```sh
npx skills add AntonLisovoy/holst-cli
```

## Использование из AI-агента

```sh
holst skill install
```

Вы выбираете агентов (Claude Code, Cursor, Codex, Copilot, Gemini или универсальную раскладку `.agents/skills`) и куда ставить навык: в текущий каталог или в профиль пользователя. Записывается одна основная копия, остальные агенты получают ссылку на нее, поэтому повторный `holst skill install` обновляет всех сразу. Без вопросов: `holst skill install -a claude-code -a cursor --project -y`.

## MCP-сервер

`holst mcp` дает агенту работать с досками через типизированные инструменты вместо команд шелла: 18 инструментов чтения (воркспейсы, доски, содержимое досок постранично и окнами текста, шаблоны, `doctor`) и 5 инструментов записи (создать, переименовать и скопировать доску; создать и изменить объекты). Инструментов удаления нет. Навык подходит, когда агент сам запускает команды `holst`; MCP подходит, когда клиент агента поддерживает MCP-серверы и нужны типизированные вызовы.

Claude Code:

```sh
claude mcp add holst -- holst mcp
```

Claude Desktop (`claude_desktop_config.json`) и Cursor (`.cursor/mcp.json`). GUI-приложения не наследуют `PATH` из шелла, поэтому укажите абсолютный путь из `which holst`:

```json
{
  "mcpServers": {
    "holst": {
      "command": "/opt/homebrew/bin/holst",
      "args": ["mcp"]
    }
  }
}
```

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.holst]
command = "/opt/homebrew/bin/holst"
args = ["mcp"]
```

Только чтение: `holst mcp --read-only` скрывает инструменты записи. Корневые флаги идут до `mcp`, `--read-only` после: `holst --base-url https://holst.example mcp --read-only`. Сервер пишет лог в stderr, клиент показывает его как лог сервера. Для живого содержимого досок нужны Node и yjs; `holst doctor` их проверит.

## Компактный вывод

Глобальные флаги идут перед командой. `--select` оставляет только перечисленные поля, `--out` пишет результат в файл и печатает короткую сводку из одной строки:

```sh
holst --select boards.id,boards.name board list
holst --out boards.json board list
```

## Автодополнение в шелле

```sh
holst completion zsh >> ~/.zshrc
```

Есть также `bash` и `fish`. Homebrew ставит дополнения для всех трех автоматически. В REPL клавиша Tab дополняет команды и опции.

## Диагностика

```sh
holst doctor
```

Проверяет версию CLI, каталог состояния, учетные данные, Node, yjs и отвечает ли API Holst, и подсказывает, что исправить. `--offline` пропускает сетевые проверки; код выхода равен 1, если какая-то проверка не прошла.

[Руководство](docs/guide.md) · [Справочник команд](docs/reference.md) · [Релизы](https://github.com/AntonLisovoy/holst-cli/releases) · [English](README.md)

Здесь размещены готовые сборки, документация и навык. Исходники хранятся
отдельно. Это неофициальный клиент Holst.
