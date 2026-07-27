# JetBrains Context: заметки после локальной проверки

Дата проверки: 28 июля 2026 года.

Это независимый отчёт о ранней версии JetBrains Context. Он не связан с
JetBrains и не содержит названий, URL, исходного кода, структуры или иной
идентифицирующей информации о проектах, использованных для проверки.

## Короткий вывод

JetBrains Context уже полезен как быстрый семантический поиск по незнакомой
кодовой базе. На трёх независимых Git-репозиториях точные запросы находили
релевантные файлы, а встроенный `doctor` полностью проверил индекс, поиск и
интеграцию с Codex.

Однако это именно early access. Настройка сложнее, чем обещает однострочный
installer, cross-repository search пока собран из нескольких команд, а условия
обработки данных требуют отдельного и очень внимательного решения.

## Что проверялось

- macOS на Apple Silicon;
- `jbcontext 0.9.5.361`;
- OpenAI Codex;
- три приватных репозитория с разными Git-историями и remote URL;
- отдельная индексация каждого репозитория;
- поиск через CLI и MCP;
- обнаружение репозиториев и один запрос по всем трём индексам;
- автоматические hooks, skill, MCP registration и execution rules для Codex.

Все три запуска `jbcontext doctor` завершились одинаково: 9 проверок пройдено,
3 пропущено. Пропуски ожидаемы — на тестовой машине настраивался только Codex,
без Claude Code, Junie и IntelliJ ACP Agents. Проверочный поиск через MCP
занимал около 0,3 секунды.

## Установка

Официальная команда:

```bash
curl -fsSL https://download.jetbrains.com/jetbrains-context/release/download-jbcontext.sh | bash
```

Затем:

```bash
jbcontext --version
jbcontext login
jbcontext setup-agent --agent=CODEX --auto --scope=USER --non-interactive
```

`setup-agent --auto` изменяет пользовательскую конфигурацию Codex:

- добавляет semantic-search skill;
- добавляет инструкции в пользовательский `AGENTS.md`;
- регистрирует MCP server;
- включает SessionStart и reminder hooks;
- добавляет execution-policy rules для read-only команд.

Перед применением полезно посмотреть план:

```bash
jbcontext setup-agent \
  --agent=CODEX \
  --auto \
  --scope=USER \
  --dump-auto \
  --print-instructions
```

## Подключение нескольких репозиториев

Репозитории индексируются отдельно. Команду нужно выполнить для каждого
рабочего дерева:

```bash
jbcontext index --project-path /path/to/repository-a
jbcontext index --project-path /path/to/repository-b
jbcontext index --project-path /path/to/repository-c
```

Проверка:

```bash
jbcontext status --project-path /path/to/repository-a
jbcontext doctor
```

Индекс привязан к Git remote и revision. Поэтому не стоит считать его
отражением всех незакоммиченных изменений рабочего дерева.

При обычной работе достаточно перейти в репозиторий:

```bash
cd /path/to/repository-a
jbcontext search \
  "where protected API requests are authenticated before business logic"
```

Когда нужная область стала известна, лучше перейти к обычному чтению файлов и
точному текстовому поиску. Повторный semantic search имеет смысл с фильтром:

```bash
jbcontext search \
  --path-filter src/backend \
  "authorization checks for protected API requests"
```

## Cross-repository workflow

В протестированной версии поиск по нескольким репозиториям является
оркестрацией, а не одной CLI-командой.

Сначала находятся точные repository IDs:

```bash
jbcontext repos "repository-a" --name-only --json-output
jbcontext repos "repository-b" --name-only --json-output
jbcontext repos "repository-c" --name-only --json-output
```

Затем один запрос выполняется отдельно и, по возможности, параллельно:

```bash
jbcontext search \
  --git-remote-url "<repository-id>" \
  --json-output \
  --limit 10 \
  "semantic description of the code being sought"
```

JetBrains публикует отдельный экспериментальный
[`org-search` skill](https://github.com/JetBrains/context/tree/main/skills/org-search),
который формализует этот процесс: discovery, выбор кандидатов, параллельный
поиск и проверка найденных файлов.

В практической проверке точные запросы внутри отдельного репозитория дали
сильные первые результаты во всех трёх случаях. Один общий запрос также
отработал во всех репозиториях, но качество ранжирования заметно различалось.
То есть multi-repo режим полезен для поиска кандидатов, но результаты всё ещё
нужно проверять по исходникам.

## Что понравилось

- Устанавливается без отдельного runtime.
- Индексация трёх разных репозиториев действительно остаётся раздельной.
- Естественные, описательные запросы находят код без знания имён символов.
- CLI возвращает путь, строки, snippet и similarity score.
- После индексации ответы приходят быстро.
- Есть JSON output для автоматизации.
- `doctor` проверяет версию, авторизацию, repository mapping, индекс, MCP и
  реальный search request.
- Codex integration устанавливает не только MCP, но и понятные инструкции о
  том, когда semantic search полезен, а когда обычный поиск быстрее.
- Заявлено, что EAP Data и поисковые запросы не используются для обучения,
  дообучения или reinforcement генеративных моделей.

## Минусы и риски

### Соглашение — самый большой минус

Перед первым использованием требуется отдельно принять
[`JetBrains Context CLI EAP User Agreement`](https://www.jetbrains.com/legal/docs/terms/jetbrains-context-cli-eap/).
Это не формальность, которую разумно молча подтверждать в installer.

Маркетинговая формулировка говорит, что исходный код не хранится на серверах.
Юридический текст заметно шире:

- JetBrains получает разрешение обрабатывать и размещать EAP Data в своей
  инфраструктуре на Google Cloud Platform;
- локально разобранные данные передаются в облако;
- векторный индекс, включая embeddings, summaries, извлечённые комментарии и
  metadata, хранится постоянно до удаления или прекращения соглашения;
- plain-text source code не должен храниться постоянно, но это уже более узкое
  обещание, чем «код не хранится»;
- EAP включает расширенный сбор диагностики, usage data и telemetry;
- продукт предоставляется «as is», может быть ненадёжным и используется на
  риск пользователя;
- общая ответственность JetBrains в большинстве случаев ограничена большей из
  двух сумм: 10 долларов США или платежей за предыдущие три месяца.

Для приватных или клиентских репозиториев решение об индексации должно
приниматься только после проверки политики компании, договоров, требований к
месту хранения данных и допустимости производных представлений кода в облаке.

### Остальные шероховатости

- Login включает browser flow, выбор лицензии и отдельное EAP-соглашение.
- Installer во время успешной установки может напечатать тревожное
  `Binary: 0B`, хотя установленная команда затем работает.
- `jbcontext repos` без точного фильтра возвращает большой общий каталог
  доступных индексов, а не компактный список только пользовательских
  репозиториев.
- Стандартный `setup-agent --auto` устанавливает обычный search skill, но не
  экспериментальный `org-search`.
- MCP `code_search` не предоставляет явный repository argument в своей схеме;
  выбор текущего проекта зависит от MCP roots. Для предсказуемого cross-repo
  поиска CLI использует отдельные вызовы с `--git-remote-url`.
- Общий абстрактный запрос может ранжироваться значительно хуже, чем точный
  запрос, сформулированный под конкретный репозиторий.
- Индексация привязана к Git revision, поэтому рабочие изменения нужно
  учитывать отдельно.
- Это EAP: интерфейс команд, hooks, skills и условия использования могут
  измениться.

## Практические рекомендации

1. Сначала прочитать EAP Agreement и решить, разрешена ли облачная индексация.
2. Перед setup выполнить `--dump-auto --print-instructions`.
3. Начать с одного некритичного репозитория.
4. Проверить `status` и `doctor`, а не полагаться только на progress output.
5. Формулировать один конкретный поведенческий вопрос за запрос.
6. После первого хорошего результата читать файл локально.
7. Для нескольких репозиториев сначала получить точные IDs через `repos`, а
   затем искать по каждому ID параллельно.
8. Не публиковать snippets, repository IDs и метрики приватных проектов в
   отчётах и issue trackers.
9. Периодически запускать `jbcontext analyze --comparison`, но не считать
   маркетинговые проценты доказанными без собственного A/B-теста.

## Полезные команды

```bash
jbcontext --help
jbcontext status
jbcontext doctor
jbcontext search --json-output --limit 10 "<focused query>"
jbcontext repos "<repository term>" --name-only --json-output
jbcontext analyze
jbcontext analyze --comparison
jbcontext remove-index
jbcontext remove-agent --agent=CODEX
jbcontext logout
```

## Источники

- [Анонс JetBrains Context](https://blog.jetbrains.com/ai/2026/07/introducing-jetbrains-context-repository-intelligence-for-coding-agents/)
- [Страница продукта и установка](https://www.jetbrains.com/context/)
- [JetBrains Context CLI EAP User Agreement](https://www.jetbrains.com/legal/docs/terms/jetbrains-context-cli-eap/)
- [Официальный репозиторий интеграций](https://github.com/JetBrains/context)
- [Исходный пост, с которого началась проверка](https://x.com/bunopus/status/2081767280275308786)
