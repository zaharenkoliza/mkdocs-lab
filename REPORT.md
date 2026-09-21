# Отчёт по лабораторной работе

## Тема

Публикация статического сайта с использованием MkDocs, GitHub Actions, GitHub Pages и сервера Helios.

## 1. Подготовка окружения

На локальном компьютере была проверена установленная версия Python. Изначально использовался Python 3.9.6, после чего была установлена актуальная версия Python 3.14.7.

Проверка версии Python и `pip`:

```bash
python3.14 --version
python3.14 -m pip --version
```

Результат:

```text
Python 3.14.7
pip 26.2.1
```

Также был установлен `virtualenv` версии 21.9.0.

## 2. Создание виртуального окружения

Был создан каталог проекта:

```bash
mkdir ~/web-research-site
cd ~/web-research-site
```

В каталоге создано виртуальное окружение:

```bash
virtualenv -p python3.14 .venv
```

Окружение активировано:

```bash
source .venv/bin/activate
```

Проверка показала, что используется Python из виртуального окружения:

```text
/Users/eozakharenko/web-research-site/.venv/bin/python
```

## 3. Установка MkDocs и фиксация зависимостей

В виртуальное окружение установлен MkDocs:

```bash
pip install mkdocs==1.6.1
```

Проверка версии:

```text
mkdocs, version 1.6.1
```

Зависимости проекта были зафиксированы в `requirements.txt`:

```bash
pip freeze > requirements.txt
```

Также был создан `.gitignore`, исключающий виртуальное окружение, результаты сборки и служебные файлы:

```text
.venv/
venv/
__pycache__/
*.py[cod]
*.egg-info/
site/
site-helios/
_build/
.DS_Store
.idea/
.vscode/
```

## 4. Создание статического сайта

Каркас сайта создан с помощью MkDocs:

```bash
mkdocs new .
```

Для лабораторной был создан нейтральный демонстрационный сайт без привязки к конкретной теме исследования.

Структура содержимого:

```text
docs/
├── index.md
├── about.md
└── tools.md
```

В `mkdocs.yml` настроены навигация, тема и встроенный поиск:

```yaml
site_name: Лабораторная работа
site_description: Публикация статического сайта с использованием MkDocs и CI/CD

theme:
  name: mkdocs
  language: ru

nav:
  - Главная: index.md
  - О работе: about.md
  - Инструменты: tools.md

use_directory_urls: true

plugins:
  - search
```

Для последующей автоматической проверки в главную страницу была добавлена контрольная строка:

```text
CONTROL_STRING_STATIC_SITE_2026
```

## 5. Проверка локальной сборки

Сайт был собран в строгом режиме:

```bash
mkdocs build --strict
```

Результат:

```text
INFO - Cleaning site directory
INFO - Building documentation to directory:
       /Users/eozakharenko/web-research-site/site
INFO - Documentation built in 0.05 seconds
```

Ошибок и предупреждений при сборке не возникло.

Наличие контрольной строки в сгенерированном HTML проверяется командой:

```bash
grep "CONTROL_STRING_STATIC_SITE_2026" site/index.html
```

## 6. Создание Git-репозитория

Локальный репозиторий был инициализирован командами:

```bash
git init
git branch -M main
```

После этого исходные файлы проекта были добавлены в Git и отправлены в удалённый репозиторий GitHub:

<https://github.com/zaharenkoliza/mkdocs-lab>

## 7. Настройка GitHub Actions и GitHub Pages

Для автоматической сборки и публикации сайта был создан workflow:

```text
.github/workflows/deploy.yml
```

Workflow выполняет:

```text
Checkout репозитория
        ↓
Установка Python
        ↓
Установка зависимостей
        ↓
mkdocs build --strict
        ↓
Создание Pages artifact
        ↓
Публикация через GitHub Pages
```

Для публикации используется официальный механизм GitHub Pages:

```text
actions/upload-pages-artifact
        +
actions/deploy-pages
```

В отличие от подхода с `peaceiris/actions-gh-pages`, этот вариант не требует публикации собранного сайта в отдельную ветку `gh-pages`: результат сборки передаётся GitHub Pages как artifact.

В настройках репозитория был выбран источник публикации:

```text
Settings → Pages → Build and deployment → Source → GitHub Actions
```

После успешного выполнения workflow сайт стал доступен по адресу:

<https://zaharenkoliza.github.io/mkdocs-lab/>

### Возникшая ошибка

При первом запуске GitHub Actions workflow завершился ошибкой на этапе конфигурации Pages:

```text
Get Pages site failed.
Please verify that the repository has Pages enabled
and configured to build using GitHub Actions.
```

Причиной было то, что GitHub Pages ещё не был включён для репозитория.

После выбора `Source = GitHub Actions` workflow был повторно запущен и завершился успешно.

> Здесь можно добавить скриншот первого неуспешного запуска и скриншот успешного выполнения workflow.

## 8. Подготовка версии для Helios

Так как сайт на Helios размещается не в корне домена, а в пользовательском подкаталоге, был создан отдельный конфигурационный файл:

```text
mkdocs-helios.yml
```

Содержимое:

```yaml
INHERIT: mkdocs.yml

site_url: https://se.ifmo.ru/~s335141/mkdocs-lab/
site_dir: site-helios
```

Отдельный `site_dir` позволяет не смешивать сборку для GitHub Pages и сборку для Helios.

Сборка для Helios выполняется командой:

```bash
mkdocs build --strict -f mkdocs-helios.yml
```

## 9. Ручное развёртывание на Helios

На сервере Helios уже существовал каталог `public_html`.

Для лабораторной был создан отдельный каталог:

```text
~/public_html/mkdocs-lab/
```

Поскольку SSH-доступ к Helios используется через порт `2222`, файлы были отправлены командой:

```bash
scp -P 2222 -r site-helios/* \
  s335141@helios.cs.ifmo.ru:~/public_html/mkdocs-lab/
```

После копирования сайт успешно открылся по адресу:

<https://se.ifmo.ru/~s335141/mkdocs-lab/>

Это подтвердило корректность конфигурации сайта для размещения в подкаталоге на Helios.
