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

[Руководство](docs/guide.md) · [Релизы](https://github.com/AntonLisovoy/holst-cli/releases) · [English](README.md)

Здесь размещены готовые сборки, документация и навык. Исходники хранятся
отдельно. Это неофициальный клиент Holst.
