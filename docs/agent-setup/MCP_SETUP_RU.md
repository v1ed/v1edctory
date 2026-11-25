# MCP: подключения для разработки и анализа GPU

Проверка источников: 25 ноября 2026 года. Все пути на Windows необходимо сверить с установленными пакетами. Конфигурации — фрагменты для слияния с действующими настройками, а не замена личного config.

## 1. Какие подключения нужны

| Подключение | Роль | Как подключать |
|---|---|---|
| Microsoft Learn | Первичная документация D3D12, DXGI, Windows | Удалённый Streamable HTTP, без auth |
| GitHub | Source, issues, PR | Официальный hosted MCP, optional при работающем `gh` |
| Nsight Graphics | Анализ NVIDIA captures, GPU traces и supported live workflows | Локальный Workflow HTTP либо Native stdio |
| PIX Timing Captures | CPU/GPU events, scheduling, I/O и memory в `.wpix` | Локальный stdio |
| WinDbg | CPU dump/live debugger context | Optional, поддержка сторонних клиентов требует проверки |

Не требуется filesystem MCP поверх уже имеющегося доступа к файлам или отдельный Git MCP поверх CLI. Собственный сервер запуска GPU tests лучше добавлять после появления надёжного CLI: сначала рабочая проверка, затем транспорт её вызова.

Удалённая модель может получать текст, который возвращают инструменты. Хранение captures на диске не гарантирует, что их извлечённые данные не попадут в контекст модели. Для работы достаточно минимальных диагностических выдержек и синтетических тестов; не прикладывайте полный игровой пакет к удалённым запросам.

## 2. Microsoft Learn

Endpoint: `https://learn.microsoft.com/api/mcp`.

Hermes:

```yaml
mcp_servers:
  microsoft_learn:
    url: "https://learn.microsoft.com/api/mcp"
```

Codex CLI:

```powershell
codex mcp add microsoft_learn --url https://learn.microsoft.com/api/mcp
codex mcp list
```

Не отправлять обычный browser GET и не делать вывод о неисправности из HTTP 405: endpoint предназначен для MCP. Проверка подключения — поиск и получение страницы, например определения `D3D12_FEATURE_DATA_D3D12_OPTIONS1`.

Источник: [Microsoft Learn MCP reference](https://learn.microsoft.com/en-us/training/support/mcp-developer-reference).

## 3. GitHub MCP

Официальный сервер: `https://api.githubcopilot.com/mcp/`, реализация [github/github-mcp-server](https://github.com/github/github-mcp-server). Существующая `gh auth login` не гарантирует автоматической авторизации отдельного HTTP MCP клиента.

Минимальная конфигурация Hermes:

```yaml
mcp_servers:
  github_read:
    url: "https://api.githubcopilot.com/mcp/"
    headers:
      Authorization: "Bearer ${GITHUB_MCP_PAT}"
      X-MCP-Toolsets: "repos,issues,pull_requests"
      X-MCP-Readonly: "true"
```

Токен задаётся в локальном `.env` профиля или через его установленный механизм credentials. Подбираются минимальные доступы для нужных repos и read-операций; не создавать административный токен для чтения source. В репозитории хранить только имя переменной. Read-only MCP ограничивает только операции этого сервера: он не запрещает `git push` через terminal.

Для Codex аналогичная настройка в `~/.codex/config.toml` или доверенном project config:

```toml
[mcp_servers.github_read]
url = "https://api.githubcopilot.com/mcp/"
bearer_token_env_var = "GITHUB_MCP_PAT"
http_headers = { "X-MCP-Toolsets" = "repos,issues,pull_requests", "X-MCP-Readonly" = "true" }
```

Codex должен получить переменную в своём процессе. Hermes `.env` не является общим хранилищем для Codex. Не предполагать поддержку hosted GitHub OAuth во всех клиентах: flow зависит от клиента/регистрации. Если `gh` уже работает, этот MCP можно не включать до появления конкретной пользы.

Источники: [server configuration](https://github.com/github/github-mcp-server/blob/main/docs/server-configuration.md), [remote server](https://github.com/github/github-mcp-server/blob/main/docs/remote-server.md), [host authentication](https://github.com/github/github-mcp-server/blob/main/docs/host-integration.md).

## 4. NVIDIA Nsight Graphics MCP

В Nsight Graphics **2026.4** документирован Windows MCP Alpha/Preview. Это реальный официальный интерфейс. Перед установкой проверить совместимые GPU/driver для используемой активности, затем записать установленную версию.

### Workflow — рекомендуемый старт

Найти `ngfx-mcp.exe` в установленном пакете, подкаталог `host/windows-desktop-nomad-x64`. Не копировать только один EXE из комплекта: host/target файлы должны оставаться вместе.

В отдельном терминале:

```powershell
$NsightMcp = Read-Host 'Полный путь к ngfx-mcp.exe из установленного Nsight Graphics'
if (-not (Test-Path -LiteralPath $NsightMcp -PathType Leaf)) { throw 'ngfx-mcp.exe не найден' }
& $NsightMcp
```

Оставить процесс запущенным. В GPU профиле Hermes:

```yaml
mcp_servers:
  nsight_workflow:
    url: "http://127.0.0.1:7661/mcp"
    supports_parallel_tool_calls: false
```

Сервер и агент должны работать на одном Windows ПК, имеющем доступ к capture. `127.0.0.1` на Mac, VPS или в облачном чате не обозначает твой Windows ПК. Remote setup требует отдельного настроенного Windows executor; это не обеспечивается одной заменой URL.

Первое сообщение агенту:

> Обнаружь инструменты и прочитай manifest Nsight. Начни с offline inspection файла по указанному абсолютному пути. Назови версию capture, первый draw/dispatch, привязанный shader и конкретное доказательство из инструмента. Не запускай приложение и не начинай live capture в этой проверке.

Для smoke test сначала получить маленький capture собственного D3D12 sample/test. Пустое подключение без рабочего capture не проверяет полноценный workflow. Если используемый shader/debugging feature недоступен на данном API/GPU, записать ограничение — MCP не добавляет отсутствующую поддержку.

### Native — дополнительный вариант

Документация также даёт `--profile native --transport stdio`. Например:

```yaml
mcp_servers:
  nsight_native:
    command: "C:/actual/Nsight Graphics/host/windows-desktop-nomad-x64/ngfx-mcp.exe"
    args: ["--profile", "native", "--transport", "stdio"]
    supports_parallel_tool_calls: false
```

Путь `C:/actual/...` является placeholder, его нужно заменить. Native и Workflow имеют разные каталоги инструментов и поведение сессий. Для первого этапа выбрать один вариант; не включать оба для параллельного управления одним capture/app.

Источники: [официальная automation documentation](https://docs.nvidia.com/nsight-graphics/UserGuide/automation.html), [2026.4 release notes](https://developer.nvidia.com/nsight-graphics/getting-started/release-note-v2026.4).

## 5. PIX Timing Captures MCP

Начиная с проверенного релиза **PIX 2610.28**, пакет содержит `PixTimingCapturesMcp.exe`. Это stdio-сервер для существующих **Timing Captures `.wpix`**. Отдельно открытый PIX UI и сетевой порт для него не нужны.

```yaml
mcp_servers:
  pix_timing:
    command: "C:/Program Files/Microsoft PIX/2610.28/PixTimingCapturesMcp.exe"
    args: []
    supports_parallel_tool_calls: false
```

Версия в пути должна совпадать с установкой. Проверить существование EXE до включения настройки. Агент запускает сервер; вручную стартовать второй экземпляр не требуется.

Первый запрос: открыть **копию** небольшого `.wpix`, перечислить доступные метрики, показать диапазон кадров, CPU/GPU events и одну подтверждаемую корреляцию. Bookmark updates и migration могут изменять файл, поэтому оригинал capture сохранить отдельно.

Этот конкретный MCP не следует объявлять универсальным GPU replay/семантическим validator. Встроенное сравнение метрик относится к двум диапазонам выбранного capture; автоматическое сравнение двух файлов не подразумевается. Для межфайлового A/B нужны одинаковые метрики и внешнее сопоставление.

В каталоге установленного PIX есть `skills`, если клиент поддерживает их загрузку. Их можно прочитать и подключить по правилам выбранного клиента; одна запись MCP не импортирует skill автоматически.

Источник: [PIX 2610.28, Analyze Timing Captures with MCP](https://devblogs.microsoft.com/pix/pix-2610-28/).

## 6. WinDbg: optional MCP

WinDbg имеет официальный MCP через `DbgX.Mcp.Proxy.exe`. Документация настройки указывает версию **1.2610.1001.0 или новее**. Официально названы VS Code с GitHub Copilot и GitHub Copilot CLI; Hermes/Codex относятся к сторонним клиентам, для которых соединение и поддержка sampling требуют отдельной проверки.

Если эта интеграция нужна:

1. Установить WinDbg из официального источника.
2. Открыть File → Settings → MCP service settings.
3. Включить Server, оставить secure mode; перед подключением target запустить MCP service.
4. В Custom посмотреть/скопировать **фактическую** proxy configuration. Не угадывать путь Microsoft Store/MSIX.
5. Перенести command/args в stdio-конфигурацию клиента.
6. Проверить `list_sessions` и подключиться к нужному process ID.

Пример только структуры:

```yaml
mcp_servers:
  windbg:
    command: "C:/actual/path/from/WinDbg/DbgX.Mcp.Proxy.exe"
    args: []
    supports_parallel_tool_calls: false
```

Одна WinDbg-сессия допускает одного MCP клиента. Если protection требует неподдерживаемого клиентом MCP sampling, интеграция считается заблокированной; сохраняем native WinDbg/экспорт анализа, а не отключаем защиту ради зелёного статуса. Эта optional интеграция не должна задерживать MS000–MS004.

Источники: [overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/windbg-mcp-overview), [setup](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/set-up-windbg-mcp).

## 7. Что пока не подключать

- **AMD RGD MCP** существует, но текущая страница требований RGD не заявляет Raphael RDNA2. Он не является обязательным инструментом для iGPU 7800X3D. [Требования RGD](https://gpuopen.com/radeon-gpu-detective/).
- **ROCm/HSA MCP/runner** не нужен для первых tests. LLVM target support не равен runtime support.
- **Ghidra MCP** можно оценить отдельно, если действительно понадобился интерактивный CPU reverse engineering. Для чтения ISA specification и Rust compiler source он не требуется.
- **Universal shell MCP** добавляет второй путь выполнения тех же команд. Для начала используются terminal и одни проверенные команды проекта.

## 8. Разделить контроль инструментов и автоматизацию

Профиль `gta6-gpu` последовательно выполняет GPU tests, game launches, PSO warm-up и live captures. `supports_parallel_tool_calls: false` не является межпроцессным lock и не блокирует другой client. Для полностью автономной работы нужна одна управляющая очередь либо named mutex/lease, соблюдаемая всеми запускателями. Установка MCP не реализует такую очередь автоматически.

После успешного CLI pipeline можно добавить небольшой локальный MCP с типизированными `list_adapters`, `run_case`, `get_result`. Это **будущая задача**, а не готовая функция данного пакета. Не делать новый MCP критической зависимостью до первого воспроизводимого теста.

## 9. Codex как дополнительный исполнитель

Для локального Codex MCP настраивается через `codex mcp add` или `~/.codex/config.toml`. Готовый фрагмент находится в `templates/codex-mcp.fragment.toml`; подключения кроме Learn отключены до проверки. Стандартные file/terminal операции уже есть.

```powershell
codex --version
codex mcp list
codex mcp --help
```

Если CLI отсутствует, использовать официальную [установку Codex](https://developers.openai.com/codex/quickstart). При npm-установке требуется установленный Node.js. Не требовать второго оркестратора: Hermes ведёт доску, а Codex может работать в назначенном отдельном worktree над одной задачей. Два клиента не должны менять одну ветку одновременно.

Использовать штатный Windows sandbox/approval режим. Если конкретному локальному инструменту не хватает разрешения, уточнить необходимое действие и доступ; не отключать защиту глобально ради GPU setup. Hosted cloud environment само по себе не даёт доступ к локальной RTX.

Источники: [Codex MCP](https://developers.openai.com/codex/mcp/), [Windows sandbox](https://developers.openai.com/codex/windows/).
