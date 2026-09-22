# Отчёт по практическому заданию P4

## P4. Развёртывание на Helios с контролем качества доставки

### Цель работы

Автоматизировать публикацию статического сайта на сервер Helios ИТМО по SSH с использованием отдельного deploy-ключа, а также реализовать:

- healthcheck после развёртывания;
- preview-сборки для отдельных веток;
- сохранение предыдущей production-версии;
- откат к предыдущей версии;
- сравнение процесса публикации на GitHub Pages и Helios.

---

## 1. Исходная конфигурация

В качестве генератора статического сайта используется MkDocs.

Основной репозиторий:

```text
https://github.com/zaharenkoliza/mkdocs-lab
```

Production-версия на GitHub Pages:

```text
https://zaharenkoliza.github.io/mkdocs-lab/
```

Production-версия на Helios:

```text
https://se.ifmo.ru/~s335141/mkdocs-lab/
```

Для Helios используется отдельный конфигурационный файл:

```text
mkdocs-helios.yml
```

Пример конфигурации:

```yaml
INHERIT: mkdocs.yml

site_url: https://se.ifmo.ru/~s335141/mkdocs-lab/
site_dir: site-helios
```

Сборка выполняется в строгом режиме:

```bash
mkdocs build --strict -f mkdocs-helios.yml
```

---

## 2. Deploy-ключ и хранение секрета

Для автоматического доступа GitHub Actions к Helios был создан отдельный SSH deploy-ключ.

Публичная часть ключа была добавлена на Helios в:

```text
~/.ssh/authorized_keys
```

Приватная часть ключа в репозиторий не добавлялась и хранится в GitHub Actions Secrets:

```text
HELIOS_SSH_KEY
```

Для подключения к Helios используется SSH-порт:

```text
2222
```

Таким образом, секрет отсутствует в исходном коде и передаётся workflow только во время выполнения job.

---

## 3. Схема автоматического развёртывания

Для Helios создан отдельный workflow:

```text
.github/workflows/deploy-helios.yml
```

Общая схема:

```text
push
  │
  ▼
checkout
  │
  ▼
setup Python
  │
  ▼
install dependencies
  │
  ▼
mkdocs build --strict
  │
  ▼
configure SSH
  │
  ├──────────── main ──────────────┐
  │                                │
  │                         backup production
  │                                │
  └──────────── branch ───────┐     │
                              │     │
                              ▼     ▼
                       preview path / production path
                              │
                              ▼
                         rsync deploy
                              │
                              ▼
                          healthcheck
```

Для копирования файлов используется `rsync` поверх SSH.

---

## 4. Развёртывание основной ветки

Основная ветка:

```text
main
```

публикуется в production-каталог:

```text
~/public_html/mkdocs-lab/
```

и доступна по адресу:

```text
https://se.ifmo.ru/~s335141/mkdocs-lab/
```

Перед каждой новой публикацией текущая production-версия сохраняется в:

```text
~/public_html/mkdocs-lab-prev/
```

Это позволяет восстановить предыдущую версию в случае ошибки новой публикации.

---

## 5. Healthcheck после развёртывания

После копирования файлов workflow выполняет автоматическую проверку опубликованного сайта.

Проверяются два условия:

1. сервер возвращает HTTP-код `200`;
2. HTML содержит контрольную строку:

```text
CONTROL_STRING_STATIC_SITE_2026
```

Логика проверки:

```bash
HTTP_CODE=$(curl \
  --silent \
  --output /tmp/site.html \
  --write-out "%{http_code}" \
  "${DEPLOY_URL}")

if [ "${HTTP_CODE}" != "200" ]; then
  echo "Expected HTTP 200, got ${HTTP_CODE}"
  exit 1
fi

if ! grep -q "CONTROL_STRING_STATIC_SITE_2026" /tmp/site.html; then
  echo "Control string not found"
  exit 1
fi
```

При несоответствии хотя бы одного условия job завершается с ошибкой.

### Проверка механизма

Первый вариант автоматического deploy успешно копировал файлы на Helios, но завершался ошибкой на шаге `Verify deployment`.

Причиной оказалось отсутствие ожидаемой контрольной строки в опубликованной версии сайта.

После добавления:

```text
CONTROL_STRING_STATIC_SITE_2026
```

в `docs/index.md` и повторной публикации healthcheck завершился успешно.

![Неуспешный healthcheck](/images/p4-healthcheck-fail.png)

![Успешный deploy на Helios](/images/p4-helios-success.png)

---

## 6. Preview-сборки

Workflow запускается не только для `main`, но и для остальных веток.

Для основной ветки используется production URL:

```text
https://se.ifmo.ru/~s335141/mkdocs-lab/
```

Для остальных веток формируется отдельный preview-каталог:

```text
~/public_html/mkdocs-lab-previews/<branch>/
```

и URL:

```text
https://se.ifmo.ru/~s335141/mkdocs-lab-previews/<branch>/
```

Для проверки была создана ветка:

```text
preview-test
```

Она была опубликована отдельно:

```text
https://se.ifmo.ru/~s335141/mkdocs-lab-previews/preview-test/
```

В preview-версии была добавлена строка, отсутствующая в production-версии.

Проверка показала, что изменения ветки `preview-test` не затронули сайт из `main`.

![Preview-сборка ветки preview-test](/images/p4-preview.png)

![Production-версия сайта](/images/p4-production.png)

---

## 7. Откат к предыдущей версии

Перед каждой новой production-публикацией предыдущая версия сохраняется в:

```text
~/public_html/mkdocs-lab-prev/
```

Для демонстрации механизма отката в основную ветку была добавлена строка:

```text
VERSION_2
```

После успешной публикации её наличие было проверено:

```bash
curl -s https://se.ifmo.ru/~s335141/mkdocs-lab/ \
  | grep "VERSION_2"
```

Затем был выполнен откат.

Команды на Helios:

```bash
rm -rf ~/public_html/mkdocs-lab-broken

mv ~/public_html/mkdocs-lab \
   ~/public_html/mkdocs-lab-broken

cp -a ~/public_html/mkdocs-lab-prev \
      ~/public_html/mkdocs-lab
```

После отката повторная проверка:

```bash
curl -s https://se.ifmo.ru/~s335141/mkdocs-lab/ \
  | grep "VERSION_2"
```

не вернула результатов.

При этом основная контрольная строка сохранилась:

```bash
curl -s https://se.ifmo.ru/~s335141/mkdocs-lab/ \
  | grep "CONTROL_STRING_STATIC_SITE_2026"
```

Таким образом, была успешно восстановлена предыдущая production-версия.

![Проверка механизма rollback](/images/p4-rollback.png)

---

## 8. Поведение при обрыве deploy

Для доставки файлов используется:

```bash
rsync -az --delete
```

в существующий production-каталог.

Такой вариант не является полностью атомарным.

Если соединение оборвётся во время передачи, возможна ситуация, при которой часть файлов уже обновлена, а остальные ещё относятся к предыдущей версии.

После передачи выполняется healthcheck. Если опубликованный сайт не соответствует ожидаемому состоянию, workflow завершается с ошибкой.

При этом предыдущая production-версия остаётся сохранённой в:

```text
~/public_html/mkdocs-lab-prev/
```

и может быть восстановлена вручную.

Более надёжный вариант дальнейшего развития:

```text
новая сборка
     ↓
временный каталог
     ↓
healthcheck
     ↓
атомарное переключение каталогов
```

В таком варианте пользователи не увидят промежуточное состояние сайта во время передачи файлов.

---

## 9. Сравнение GitHub Pages и Helios

| Критерий | GitHub Pages | Helios |
|---|---|---|
| Способ доставки | Pages artifact | SSH + rsync |
| Production deploy | автоматический | автоматический |
| Доступ к серверу | не требуется | требуется SSH |
| Healthcheck | публикация контролируется платформой; дополнительная проверка возможна отдельно | реализован собственный HTTP 200 + контрольная строка |
| Preview веток | в данной работе отдельно не настраивался | реализован отдельный каталог для каждой ветки |
| Откат | повторный deploy нужного состояния Git | сохранение `mkdocs-lab-prev` |
| Управление файлами | скрыто платформой | полный доступ к структуре каталогов |
| Отладка | GitHub Actions и Pages | GitHub Actions + SSH |
| Риск частичного обновления | инфраструктура скрыта платформой | возможен при обрыве `rsync` |
| Время доставки | **37 с** | **34 с** |

Для сравнения использован один и тот же push/commit `25ab9fa` (`Add rollback test version`), чтобы условия были одинаковыми:

- GitHub Pages — **37 с**;
- Helios — **34 с**.

На этом запуске Helios оказался на 3 секунды быстрее. По другим запускам время менялось, поэтому эти значения следует рассматривать как измерение конкретного запуска, а не как устойчивую характеристику платформ.

### Наблюдения

GitHub Pages требует меньше собственной инфраструктуры и предоставляет более высокий уровень абстракции.

Helios потребовал ручной настройки SSH-доступа, структуры каталогов, healthcheck, preview-сборок и rollback, но при этом дал больший контроль над процессом доставки.

---

## 10. Результат

В ходе выполнения P4 была реализована автоматическая публикация MkDocs-сайта на Helios ИТМО.

Реализованы:

- отдельный SSH deploy-ключ;
- хранение приватного ключа в GitHub Actions Secrets;
- автоматическая сборка сайта;
- доставка по SSH/rsync;
- production-развёртывание из основной ветки;
- preview-сборки для остальных веток;
- healthcheck после публикации;
- проверка HTTP-кода `200`;
- проверка контрольной строки;
- сохранение предыдущей production-версии;
- демонстрация rollback;
- анализ поведения при прерывании deploy;
- сравнение Helios с GitHub Pages.

Практическое задание P4 выполнено.
