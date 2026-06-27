# Free Google Disk Backup

Бесплатная схема регулярных production-бэкапов на Google Drive:
**Restic + rclone + Google Drive + systemd + Telegram**.

Проект нужен, когда хочется получить нормальные зашифрованные бэкапы без
платного Hetzner Backup, S3, Google Cloud Storage или отдельного backup-SaaS.
Используется уже имеющееся место в Google Drive.

> Важно: Google Cloud project здесь нужен только для Google Drive API и OAuth.
> Эта схема не использует Google Cloud Storage, Compute Engine, Cloud SQL,
> BigQuery или другие платные GCP-сервисы.

## Зачем сделан этот репозиторий

Этот репозиторий родился из практической проблемы vibe coding. Когда продукт
активно собирается с помощью AI-агентов, скорость разработки становится
огромной, но появляется новый риск: агент может ошибиться, перезаписать файл,
сломать миграцию, удалить данные, неудачно применить deploy-команду или сделать
изменение, которое заметят только через несколько недель.

Git защищает исходный код, но не всегда защищает production-данные:

- базу данных;
- production `.env`;
- пользовательские файлы;
- OCR/images/media;
- состояние, которое живёт только на сервере.

Цель этого Skill/README — дать независимый safety layer для AI-first проектов:
бэкапы должны жить отдельно от сервера, отдельно от GitHub и отдельно от
текущего агента. Если AI случайно что-то снесёт, должна быть понятная точка
восстановления.

Почему Google Drive: у многих solo-builders и AI-подписчиков уже есть много
места в Google Drive вместе с Google One / AI Pro / AI Plus / Workspace. Это
место можно использовать для encrypted backup repository, не покупая отдельное
backup-хранилище.

Бэкапить через Restic/rclone — не новая идея. Но этот репозиторий делает
понятный safety kit для AI-first разработки: с готовым Skill, человеческим
runbook, Telegram-уведомлениями, restore-test и акцентом на том, что backup
должен быть восстановимым, а не просто “куда-то загруженным”.

## Что даёт эта схема

- Зашифрованные бэкапы: Google Drive видит только encrypted Restic repository.
- Гибридная глубина хранения:
  - почасовые бэкапы за 48 часов;
  - дневные бэкапы за 30 дней;
  - недельные бэкапы за 12 недель;
  - месячные бэкапы за 12 месяцев.
- Telegram-уведомления об успешных и неуспешных бэкапах.
- Daily summary: последний snapshot, состояние repository, restore-test,
  maintenance, место на Google Drive.
- Restore-first подход: бэкап считается рабочим только после test restore.
- Понятный handoff для будущего инженера или AI-агента.

## Quick Start за 5 минут

1. Выбери code system, где будешь ставить backup skill/rule.
2. Скопируй нужный adapter из `agent-adapters/` в место установки.
3. Вызови skill/rule командой из таблицы ниже.
4. Дай агенту доступы только по необходимости: SSH, GitHub, Google OAuth,
   Telegram bot token.
5. Не вставляй в чат Restic password, OAuth client secret, Telegram token,
   `.env` или `rclone.conf`.
6. Не считай задачу готовой, пока агент не покажет успешный restore-test.

Минимальный prompt после установки:

```text
Use $free-google-disk-backup.

Set up encrypted regular backups for this production project to my existing
Google Drive using Restic + rclone. Do not use paid backup storage. Do not
delete anything without explicit approval. Do not print secrets. Prove restore
works before calling the setup complete.
```

Для Claude Code/Grok Build с Claude-compatible skills вместо первой строки
используй:

```text
/free-google-disk-backup
```

## Compatibility Matrix

| Code system | Status | Adapter | Invocation |
| --- | --- | --- | --- |
| Codex | Supported | `agent-adapters/codex-skill/free-google-disk-backup` | `Use $free-google-disk-backup.` |
| Claude Code | Supported | `agent-adapters/claude-code-skill/free-google-disk-backup` | `/free-google-disk-backup` |
| Qwen Code | Supported | `agent-adapters/qwen-code-skill/free-google-disk-backup` | `Use free-google-disk-backup.` or `/skills` |
| Cursor | Supported as Rules | `agent-adapters/cursor-rules/.cursor/rules/free-google-disk-backup.mdc` | `Use the Free Google Disk Backup rule.` |
| Grok Build | Conditional | Use Claude Code adapter only if your Grok Build terminal supports Claude Code skills | `/free-google-disk-backup` |
| Google Antigravity | Research needed | No installable adapter yet | Use this README manually |
| Z.ai / GLM | Research needed | No installable adapter yet | Use this README manually |

## No Restore Test = No Backup

Backup upload is not proof. A backup is useful only if it can be restored.

Before declaring success, the agent must show evidence for at least:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh stage latest
sudo /opt/<project>/ops/backup/restore.sh test latest
```

For important production projects, prefer a Stage Restore Server / restore-lab
instead of testing restore on the production server.

The handoff must record:

- latest verified snapshot ID;
- restore-test timestamp;
- where the Restic password file lives;
- where `rclone.conf` lives;
- exact restore commands;
- what was restored and inspected.

## Архитектура

```text
Production server
  |
  |-- backup.sh
  |     |-- database dump
  |     |-- .env/secrets copy
  |     |-- durable user files
  |     |-- deploy files
  |     |-- manifest.json
  |     |-- checksums.sha256
  |
  |-- restic encrypts/deduplicates/compresses
  |
  |-- rclone writes encrypted repository
  v
Google Drive: <ProjectName>-Backups/restic

systemd timers run backup/summary/maintenance/restore-test
Telegram bot reports status
```

## Что обычно бэкапить

Включать:

- database logical dump;
- production `.env` или другой secrets/config file;
- пользовательские загрузки;
- OCR/images/documents/media, если их нельзя пересоздать;
- `docker-compose.yml`;
- `Caddyfile` или nginx config;
- Git bundle или точный Git commit/source metadata;
- `manifest.json`;
- `checksums.sha256`.

Не включать, если нет особой причины:

- Redis cache/queue;
- Docker images;
- `node_modules`;
- virtualenv;
- build artifacts;
- временные файлы;
- логи.

## Типовая структура на сервере

```text
/opt/<project>/ops/backup/          backup scripts in repo
/etc/<project>-backup/              root-only config and secrets
/var/lib/<project>-backup/          state/cache/staging
/var/lib/<project>-restore/         staged restores
```

Критически важный файл:

```text
/etc/<project>-backup/restic-password
```

Без Restic password восстановить данные из Google Drive нельзя. Файлы останутся
зашифрованными. Этот пароль нужно сохранить в password manager.

## Самое важное: не потерять Restic password

Restic password — это главный ключ ко всем бэкапам. Google Drive хранит
зашифрованный Restic repository. Это хорошо для безопасности, но означает
жёсткое правило:

> Если потерять Restic password, восстановить данные из Google Drive нельзя.

Файл обычно лежит здесь:

```text
/etc/<project>-backup/restic-password
```

Проверить, что файл существует и закрыт правами:

```bash
sudo ls -l /etc/<project>-backup/restic-password
```

Хороший результат должен выглядеть примерно так:

```text
-rw------- 1 root root ... /etc/<project>-backup/restic-password
```

Проверить, что в файле есть содержимое, но не выводить пароль:

```bash
sudo wc -c /etc/<project>-backup/restic-password
```

Чтобы сохранить пароль в password manager, открой его только на своём сервере
и не вставляй в чат, GitHub issue, README, логи или prompt:

```bash
sudo cat /etc/<project>-backup/restic-password
```

После этого сразу сохрани значение в password manager с названием вроде:

```text
<ProjectName> Restic Backup Password
```

В handoff-документе нужно писать только путь к файлу, но не сам пароль.

## Google Cloud: подробная настройка Drive API

Google Cloud Console не всегда очевиден: Google периодически меняет названия
разделов, а Google Auth Platform может выглядеть иначе у разных аккаунтов.
Ниже практический путь, который обычно работает.

### 1. Создать или выбрать Google Cloud project

1. Открой [Google Cloud Console](https://console.cloud.google.com/).
2. В верхней панели выбери текущий project или нажми project selector.
3. Нажми **New Project**.
4. Назови project, например:

```text
Free Google Disk Backup
```

или под конкретный проект:

```text
CashbackBot Backups
```

5. Нажми **Create**.
6. Убедись, что в верхней панели выбран именно этот project.

Официальная документация Google: [Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project).

### 2. Включить Google Drive API

Есть два рабочих пути.

Вариант A, через меню:

1. Открой [Google Cloud Console](https://console.cloud.google.com/).
2. Проверь, что выбран нужный project.
3. Открой левое меню.
4. Перейди в **APIs & Services**.
5. Открой **Library**.
6. В поиске введи:

```text
Google Drive API
```

7. Открой карточку **Google Drive API**.
8. Нажми **Enable**.

Вариант B, через прямую ссылку:

1. Открой [API Library](https://console.cloud.google.com/apis/library).
2. Проверь project в верхней панели.
3. Найди **Google Drive API**.
4. Нажми **Enable**.

Официальная документация Google говорит включать API через
**APIs & Services > Library**, выбрать нужный API и нажать **Enable**:
[Enable Google Workspace APIs](https://developers.google.com/workspace/guides/enable-apis).

### 3. Настроить OAuth consent / Google Auth Platform

1. В Google Cloud Console открой **Google Auth Platform**.
2. Если такого пункта не видно, попробуй:
   - **APIs & Services > OAuth consent screen**;
   - поиск по консоли: `OAuth consent`;
   - поиск по консоли: `Google Auth Platform`.
3. Выбери user type:
   - **External**, если это личный Gmail или приложение не внутри Workspace
     organization;
   - **Internal**, если это Google Workspace project только для организации.
4. Заполни app name, user support email и developer contact email.
5. Сохрани.

Для личного проекта обычно достаточно режима **Testing** или **In production**.
Если приложение только для твоего собственного аккаунта и rclone, публичная
верификация Google обычно не нужна.

Официальная страница: [Configure OAuth consent](https://developers.google.com/workspace/guides/configure-oauth-consent).

### 4. Добавить scope `drive.file`

В Google Auth Platform:

1. Открой **Data Access**.
2. Нажми **Add or remove scopes**.
3. Найди или вручную добавь:

```text
https://www.googleapis.com/auth/drive.file
```

4. Сохрани изменения.

Почему именно `drive.file`:

- rclone сможет читать/изменять только файлы, созданные или открытые этим
  приложением;
- это меньше прав, чем полный `drive`;
- это хорошо подходит для backup repository.

rclone тоже описывает `drive.file` как scope, при котором rclone работает с
созданными им файлами и не видит обычные файлы пользователя в Drive:
[rclone Google Drive scopes](https://rclone.org/drive/#scopes).

### 5. Добавить test user, если приложение в Testing

Если publishing status = **Testing**, Google пустит только test users.

1. Открой **Google Auth Platform > Audience**.
2. Найди блок test users.
3. Добавь Gmail-адрес, под которым будешь проходить rclone OAuth.
4. Сохрани.

Если Google пишет `Ineligible accounts not added`, значит указанный email не
подходит как Google Account или введён не тот адрес. Используй тот Gmail, под
которым реально открывается Google Drive.

### 6. Создать OAuth Client ID

1. Открой **Google Auth Platform > Clients**.
2. Нажми **Create Client**.
3. Application type:

```text
Desktop app
```

4. Название, например:

```text
rclone backup client
```

5. Создай client.
6. Сохрани:
   - Client ID;
   - Client secret.

Не коммить эти значения в GitHub и не отправляй их в чат. Их нужно вводить в
rclone или хранить в root-only config на сервере.

Официальная документация Google: [Create access credentials](https://developers.google.com/workspace/guides/create-credentials).

### 7. Настроить rclone

На сервере или локально:

```bash
rclone config
```

Примерные ответы:

```text
n) New remote
name> gdrive
Storage> drive
client_id> <OAuth Client ID>
client_secret> <OAuth Client secret>
scope> drive.file
root_folder_id> оставь пустым
service_account_file> оставь пустым
Use web browser to automatically authenticate rclone with remote?
```

Если rclone запускается на машине с браузером, можно выбрать `Y`.

Если rclone запускается на сервере без браузера, обычно выбирают `N`, а rclone
даёт команду или ссылку для browser-assisted auth. Иногда ссылка ведёт на
`127.0.0.1`. Это нормально: rclone временно поднимает локальный callback server
для OAuth. Если встроенный браузер не открывает `127.0.0.1`, открой ссылку в
обычном браузере на той машине, где запущен callback, или используй manual auth.

Успешный результат выглядит примерно так:

```text
Success!
All done. Please go back to rclone.
```

После этого проверить:

```bash
rclone lsd gdrive:
```

Официальная документация rclone: [Google Drive backend](https://rclone.org/drive/).

## Restic repository

Пример repository path:

```text
rclone:gdrive:MyProject-Backups/restic
```

Restic password хранится только на сервере и в password manager:

```text
/etc/<project>-backup/restic-password
```

Проверка snapshot list:

```bash
RESTIC_PASSWORD_FILE=/etc/<project>-backup/restic-password \
restic -r rclone:gdrive:MyProject-Backups/restic snapshots
```

## systemd timers

Рекомендуемые таймеры:

```text
<project>-backup.timer              hourly backup
<project>-backup-summary.timer      daily Telegram summary
<project>-backup-maintenance.timer  weekly retention/prune/check
<project>-backup-full-check.timer   monthly repository check
<project>-backup-restore-test.timer monthly isolated restore test
```

Проверка:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
sudo systemctl status <project>-backup.service --no-pager -l
sudo journalctl -u <project>-backup.service -n 100 --no-pager
```

## Как поменять частоту бэкапов и проверок

Расписание задаётся в systemd `.timer` файлах через `OnCalendar`.

Важно различать:

- `backup.timer` — создаёт новые backup snapshots;
- `maintenance.timer` — чистит старые snapshots по retention и проверяет
  repository;
- `restore-test.timer` — пробует восстановить backup в isolated test mode.

Если хочется “чаще бэкапить”, меняй `backup.timer`. Если хочется чаще чистить и
проверять repository, меняй `maintenance.timer`. Если хочется чаще доказывать,
что восстановление работает, меняй `restore-test.timer`.

Сначала посмотри текущие таймеры:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
```

Посмотреть содержимое конкретного timer:

```bash
sudo systemctl cat <project>-backup.timer --no-pager
sudo systemctl cat <project>-backup-maintenance.timer --no-pager
sudo systemctl cat <project>-backup-restore-test.timer --no-pager
```

### Пример: hourly backup чаще или реже

Каждый час:

```ini
OnCalendar=hourly
```

Каждые 30 минут:

```ini
OnCalendar=*:0/30
```

Каждые 6 часов:

```ini
OnCalendar=00/6:00
```

### Пример: maintenance 3 раза в неделю

Если weekly maintenance нужно запускать не только по воскресеньям, а 3 раза в
неделю, например понедельник/среда/пятница в 03:30 UTC:

```ini
OnCalendar=Mon,Wed,Fri *-*-* 03:30:00
RandomizedDelaySec=10m
```

Это удобно, если проект быстро меняется и хочется чаще делать `restic forget`,
`prune` и repository check.

### Пример: restore-test 2 раза в месяц

Например 1-го и 15-го числа каждого месяца в 05:30 UTC:

```ini
OnCalendar=*-*-01,15 05:30:00
RandomizedDelaySec=30m
```

Такой timer будет дважды в месяц проверять, что snapshot действительно
восстанавливается в isolated test environment.

### Как безопасно изменить timer

Предпочтительно редактировать override, а не основной unit-файл:

```bash
sudo systemctl edit <project>-backup-maintenance.timer
```

Пример override для 3 раз в неделю:

```ini
[Timer]
OnCalendar=
OnCalendar=Mon,Wed,Fri *-*-* 03:30:00
RandomizedDelaySec=10m
```

Пустая строка `OnCalendar=` нужна systemd, чтобы сбросить старое расписание
перед заданием нового.

После изменения:

```bash
sudo systemctl daemon-reload
sudo systemctl restart <project>-backup-maintenance.timer
sudo systemctl list-timers '*backup*' --all --no-pager
```

Проверь колонку `NEXT`: там должно быть новое ожидаемое время запуска.

## Telegram bot

Лучше создать отдельного backup-бота через BotFather, а не использовать
production-бота проекта.

Нужно сохранить на сервере:

```text
TELEGRAM_BACKUP_BOT_TOKEN=<secret>
TELEGRAM_BACKUP_CHAT_ID=<chat id>
```

Не хранить токен в GitHub и не писать его в чат.

## Как читать daily summary

Пример:

```text
Cashback backup daily summary
Latest: cc182e05... (OK)
Timestamp: 2026-06-18T08:03:06Z
Duration: 41s
Repository: ok
Maintenance: missing
Restore test: ok
Google Drive used/free: 5063760/5496985770348
```

Значение:

- `Latest (OK)` — последний backup snapshot создан успешно.
- `Repository: ok` — Restic repository доступен и читается.
- `Maintenance: missing` — weekly maintenance ещё не запускался или state-файл
  ещё не создан. До первого weekly запуска это нормально.
- `Restore test: ok` — isolated restore test прошёл.
- `Google Drive used/free` — занято/свободно в байтах.

Если `Maintenance: missing` остаётся после первого weekly maintenance, нужно
проверить:

```bash
sudo systemctl status <project>-backup-maintenance.timer --no-pager -l
sudo systemctl status <project>-backup-maintenance.service --no-pager -l
sudo journalctl -u <project>-backup-maintenance.service -n 100 --no-pager
```

## Restore-first правило

Backup не считается готовым, пока не прошёл restore-test.

Самый безопасный вариант для серьёзных проектов — проверять восстановление не
на production server, а на отдельном Stage Restore Server / restore-lab. Это
отдельный VPS или test server, который читает Google Drive Restic repository,
восстанавливает snapshot, поднимает isolated database/container и доказывает,
что backup можно восстановить без касания production.

Подробный сценарий: [docs/stage-restore-server.md](docs/stage-restore-server.md).

Минимальный безопасный workflow:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh stage latest
sudo /opt/<project>/ops/backup/restore.sh test latest
```

Что делает безопасный test restore:

- берёт snapshot из Restic repository;
- разворачивает его во временную/staging область;
- поднимает isolated database/container, если это предусмотрено скриптом;
- пробует восстановить database dump;
- проверяет ключевые таблицы/файлы;
- не трогает production database;
- не заменяет production `.env`;
- не останавливает production-сервисы.

Если хочется не доверять daily summary, а проверить руками:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
```

Хороший результат в state-файле:

```json
{
  "status": "ok",
  "snapshot_id": "<snapshot-id>",
  "timestamp": "<timestamp>",
  "duration_seconds": 47
}
```

Если нужно посмотреть содержимое backup без production restore:

```bash
sudo /opt/<project>/ops/backup/restore.sh stage latest
```

После staging можно проверять `manifest.json`, `checksums.sha256`, наличие
database dump и нужных файлов. Секреты вроде `.env` можно проверять по факту
наличия, но не печатать в чат.

Production restore должен быть защищён:

- только по явному разрешению владельца;
- только с конкретным snapshot ID;
- только после успешного isolated restore-test;
- желательно с confirmation token вроде
  `RESTORE_CONFIRMATION=RESTORE_<PROJECT>_PRODUCTION`.

## Готовый prompt для AI-агента

```text
Use $free-google-disk-backup.

Мне нужно организовать бесплатные регулярные бэкапы этого production-проекта
на мой Google Drive. Используй Restic + rclone. Не используй платное хранилище
Hetzner, Google Cloud Storage, S3 или другие платные backup-сервисы.
Ничего не удаляй без явного разрешения.

Нужна гибридная глубина:
- почасовые бэкапы за 48 часов
- дневные бэкапы за 30 дней
- недельные бэкапы за 12 недель
- месячные бэкапы за 12 месяцев

Нужно бэкапить базу данных, production .env/secrets, пользовательские файлы,
важные deploy-файлы и source metadata/Git bundle. Не бэкапить кеши,
Docker images, зависимости и пересобираемые артефакты, если это не бизнес-данные.

Добавь Telegram-уведомления, daily summary, weekly maintenance и restore-test.
Оставь подробный handoff-документ: где лежит Restic password, rclone config,
как посмотреть статус, как расшифровать и восстановить бэкап.
```

## Структура этого пакета

```text
free-google-disk-backup/
  README.md
  LICENSE
  SECURITY.md
  CONTRIBUTING.md
  CHANGELOG.md
  ROADMAP.md
  docs/
    FreeGoogleDiskBackup.md
    agent-compatibility.md
    checklists.md
    handoff-template.md
    publishing-checklist.md
    restore-drill.md
    stage-restore-server.md
    troubleshooting.md
    threat-model.md
  examples/
    backup.env.example
    handoff/
      example-handoff.md
    prompts/
      README.md
      codex.md
      claude-code.md
      cursor.md
      grok-build-compat.md
      qwen-code.md
    telegram-summary.txt
    systemd/
  agent-adapters/
    README.md
    codex-skill/
      README.md
      free-google-disk-backup/
        SKILL.md
        agents/
          openai.yaml
        references/
          human-runbook.md
    claude-code-skill/
      README.md
      free-google-disk-backup/
        SKILL.md
        references/
          human-runbook.md
    qwen-code-skill/
      README.md
      free-google-disk-backup/
        SKILL.md
        references/
          human-runbook.md
    cursor-rules/
      README.md
      .cursor/
        rules/
          free-google-disk-backup.mdc
    research-needed.md
  .github/
    ISSUE_TEMPLATE/
      bug_report.md
      feature_request.md
    PULL_REQUEST_TEMPLATE.md
```

## Установка в Codex

Для Codex используй адаптер:

```text
agent-adapters/codex-skill/free-google-disk-backup
```

Скопируй его в:

```text
~/.codex/skills/free-google-disk-backup
```

После этого в новом чате можно писать:

```text
Use $free-google-disk-backup.
```

Если хочешь установить из GitHub в новом Codex-чате:

```text
Install the Codex skill from this repository:
<GITHUB_REPO_URL>

Use agent-adapters/codex-skill/free-google-disk-backup and copy it to
~/.codex/skills/free-google-disk-backup. Do not copy secrets.
```

## Установка в Claude Code

Claude Code тоже поддерживает skills: папка skill-а содержит `SKILL.md`, а
личные skills лежат в `~/.claude/skills/<skill-name>/`. Подробности:
[Claude Code skills docs](https://code.claude.com/docs/en/skills).

Для Claude Code используй адаптированную папку:

```text
agent-adapters/claude-code-skill/free-google-disk-backup
```

Скопируй её в:

```text
~/.claude/skills/free-google-disk-backup
```

После этого в Claude Code можно вызывать:

```text
/free-google-disk-backup
```

Если хочешь установить из GitHub в новом Claude Code чате, можно написать:

```text
Install the Claude Code skill from this repository:
<GITHUB_REPO_URL>

Use agent-adapters/claude-code-skill/free-google-disk-backup and copy it to
~/.claude/skills/free-google-disk-backup. Do not copy secrets.
```

## Установка в Qwen Code

Qwen Code поддерживает Agent Skills: личные skills лежат в
`~/.qwen/skills/<skill-name>/`, а project-local skills — в
`.qwen/skills/<skill-name>/`. Подробности:
[Qwen Code Agent Skills](https://qwenlm.github.io/qwen-code-docs/en/users/features/skills/).

Для Qwen Code используй адаптированную папку:

```text
agent-adapters/qwen-code-skill/free-google-disk-backup
```

Скопируй её в:

```text
~/.qwen/skills/free-google-disk-backup
```

В Qwen Code можно открыть список skills через:

```text
/skills
```

После установки попроси Qwen Code использовать skill:

```text
Use free-google-disk-backup.
```

## Установка в Cursor

Cursor использует **Rules**, а не Skills. Поэтому для Cursor в репозитории
лежит отдельный адаптер:

```text
agent-adapters/cursor-rules/.cursor/rules/free-google-disk-backup.mdc
```

Скопируй этот файл в проект, где Cursor должен учитывать правила бэкапа:

```text
<project>/.cursor/rules/free-google-disk-backup.mdc
```

Подробности: [Cursor Rules docs](https://cursor.com/docs/rules).

## Поддерживаемые code systems и вызов

| Code system | Тип адаптера | Путь в этом репозитории | Как вызвать после установки |
| --- | --- | --- | --- |
| Codex | Skill | `agent-adapters/codex-skill/free-google-disk-backup` | `Use $free-google-disk-backup.` |
| Claude Code | Skill | `agent-adapters/claude-code-skill/free-google-disk-backup` | `/free-google-disk-backup` |
| Qwen Code | Agent Skill | `agent-adapters/qwen-code-skill/free-google-disk-backup` | `Use free-google-disk-backup.` или открыть `/skills` |
| Cursor | Project Rule | `agent-adapters/cursor-rules/.cursor/rules/free-google-disk-backup.mdc` | Написать в чате: `Use the Free Google Disk Backup rule.` |

Важно: Cursor adapter — это не slash-команда и не Skill, а Project Rule.

Если Grok Build в твоём терминале реально читает Claude Code skills, используй
Claude Code adapter и вызывай так же:

```text
/free-google-disk-backup
```

## Grok Build, Google Antigravity, Z.ai / GLM

Для этих инструментов отдельный installable adapter пока не добавлен намеренно.
Я не нашёл проверенного публичного формата Skill/Rules для:

- Grok Build;
- Google Antigravity;
- Z.ai / GLM coding agent setup.

Если Grok Build в твоём терминале действительно умеет читать Claude Code
skills, используй:

```text
agent-adapters/claude-code-skill/free-google-disk-backup
```

То есть не нужна отдельная папка `grok-build`, пока официальный формат не
отличается от Claude Code.

Чтобы не придумывать несуществующий стандарт, статус вынесен в
[agent-adapters/research-needed.md](agent-adapters/research-needed.md). Пока
для них можно использовать корневой README как prompt/runbook вручную.

## Документы в репозитории

- [SECURITY.md](SECURITY.md) — правила работы с секретами и threat model.
- [CONTRIBUTING.md](CONTRIBUTING.md) — как предлагать улучшения и не утекать
  секретами в PR.
- [ROADMAP.md](ROADMAP.md) — идеи дальнейшего развития.
- [CHANGELOG.md](CHANGELOG.md) — история версий.
- [docs/checklists.md](docs/checklists.md) — pre-install, verification,
  weekly maintenance и monthly restore drill чеклисты.
- [docs/agent-compatibility.md](docs/agent-compatibility.md) — поддерживаемые
  code systems, invocation и политика добавления новых adapters.
- [docs/handoff-template.md](docs/handoff-template.md) — шаблон handoff-файла
  для конкретного production-проекта.
- [docs/publishing-checklist.md](docs/publishing-checklist.md) — проверка
  перед публикацией репозитория в open source.
- [docs/restore-drill.md](docs/restore-drill.md) — как регулярно проводить
  учебное восстановление и что записывать как доказательство.
- [docs/stage-restore-server.md](docs/stage-restore-server.md) — preferred
  restore-lab сценарий на отдельном stage/test server.
- [docs/troubleshooting.md](docs/troubleshooting.md) — типовые проблемы
  Google OAuth, rclone, Restic, Telegram и restore-test.
- [docs/threat-model.md](docs/threat-model.md) — от чего схема защищает и от
  чего не защищает.
- [examples/backup.env.example](examples/backup.env.example) — пример backup
  config без реальных секретов.
- [examples/prompts/](examples/prompts/) — готовые prompts для Codex, Claude
  Code, Qwen Code, Cursor и Grok Build compatibility mode.
- [examples/handoff/example-handoff.md](examples/handoff/example-handoff.md) —
  пример заполненного handoff без секретов.
- [examples/systemd/](examples/systemd/) — примеры systemd service/timer units,
  включая stage restore-lab timer.
- [agent-adapters/](agent-adapters/) — installable adapters для Codex,
  Claude Code, Qwen Code и Cursor Rules.
- [.github/](.github/) — issue templates и PR template с secret-safety
  checklist.

## FAQ

### Это правда бесплатно?

Схема не использует платное backup-хранилище. Но она расходует место в твоём
Google Drive. Если Google Drive уже оплачен или имеет достаточно бесплатного
места, дополнительных storage-расходов быть не должно.

Для спокойствия можно поставить Google Cloud Billing budget alert на `$1`.
Budget alert предупреждает, но не останавливает расходы автоматически.

### Google не забанит аккаунт за rclone?

rclone — обычный open-source инструмент, который использует OAuth и Google Drive
API. Важно не нарушать лимиты, ToS и не делать подозрительную массовую
активность. Почасовой Restic backup небольшого проекта обычно выглядит как
обычное API-использование.

### Почему не Google Cloud Storage?

Потому что цель этой схемы — не платить за отдельное объектное хранилище.
Google Cloud Storage технически хорош, но это уже другой, потенциально платный
вариант.

### Почему нужен отдельный OAuth client?

Так проще контролировать квоты, доступы и безопасность. Не нужно зависеть от
shared rclone client.

### Почему `drive.file`, а не полный `drive`?

`drive.file` даёт меньше прав: rclone работает с файлами, которые созданы или
открыты этим приложением. Для backup repository этого достаточно и безопаснее.

### Что будет, если потерять Restic password?

Бэкапы на Google Drive останутся зашифрованными. Восстановить данные без пароля
нельзя. Это не формальность, это главный секрет всей системы.

## Источники

- Google: [Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project)
- Google: [Enable Google Workspace APIs](https://developers.google.com/workspace/guides/enable-apis)
- Google: [Configure OAuth consent](https://developers.google.com/workspace/guides/configure-oauth-consent)
- Google: [Create access credentials](https://developers.google.com/workspace/guides/create-credentials)
- rclone: [Google Drive backend](https://rclone.org/drive/)
- Claude Code: [Skills](https://code.claude.com/docs/en/skills)
- Qwen Code: [Agent Skills](https://qwenlm.github.io/qwen-code-docs/en/users/features/skills/)
- Cursor: [Rules](https://cursor.com/docs/rules)
- Google: [Antigravity](https://antigravity.google/)
