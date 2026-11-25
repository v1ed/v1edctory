# Пошаговая подготовка VIctory к разработке с агентами

Дата проверки: **25.11.2026**. Это инструкция для Windows 11 и конфигурации Ryzen 7 7800X3D + AMD iGPU + RTX 5070. Объект проверки — оригинальный [Marvisak/VIctory](https://github.com/Marvisak/VIctory), коммит `940d32f69b22265690a0b5a15d6ded572ba33baf`. Он является воспроизводимой отправной точкой, а не объявленной стабильной сборкой.

## 0. Принять границы первого этапа

Ближайшая цель — воспроизводимая сборка, проверенный выбор адаптера, честная отчётность тестов и небольшой набор независимых синтетических проверок. Затем — остановка на неподдерживаемой операции с достаточной диагностикой. Получение 30 FPS не является критерием готовности окружения и не имеет подтверждённого срока.

Для быстрого старта сохраняем существующие Rust, D3D12, SSA и прямой DXIL emitter. Новый GPU Command IR, полный CPU-интерпретатор RDNA2, Vulkan backend и собственный MCP пока не нужны. Изменение архитектуры должно устранять конкретную измеренную проблему.

Для этой последовательности выбран явный этап допуска: до приёмки MS003 игровые файлы анализируются офлайн, игра через `run --execute` не запускается ради baseline или capture. Синтетические тесты, создание PSO и captures собственных минимальных приложений допустимы. Это инженерная политика старта с требованием без молчаливых пропусков, а не ограничение возможностей upstream.

Полный аудит прежних утверждений находится в [VERIFICATION_AUDIT_RU.md](VERIFICATION_AUDIT_RU.md). Особенно важны значения `--strict`, пропускаемые tests и ограничения shader loop budget.

## 1. Установить минимальный набор инструментов

Если инструмент уже работает, сначала записать его версию; повторная установка не требуется.

| Инструмент | Зачем | Обязательность |
|---|---|---|
| Git for Windows | Форк, ветки, worktree, история | Обязательно |
| GitHub CLI `gh` | Авторизация, форк, PR, чтение issues | Рекомендуемый основной GitHub интерфейс |
| PowerShell 7 | Единая среда команд и сценариев | Рекомендуется |
| Visual Studio / Build Tools с MSVC x64 и Windows SDK | Линковка Windows Rust и native тестов | Обязательно |
| Rustup | Установка toolchain из репозитория | Обязательно |
| `rustfmt`, `clippy` | Уже перечислены в toolchain | Обязательно для существующих проверок |
| Современный Microsoft DXC package x64 | `dxil.dll` validator; `dxc.exe` для эталонных HLSL | Обязательно для полного Windows shader test setup |
| Драйверы NVIDIA и AMD | Доступ к обоим GPU | Обязательно |
| Windows Graphics Tools | D3D12 Debug Layer и GBV | Для GPU QA |
| PIX | Захваты D3D12, анализ CPU/GPU и timing MCP | Для графической диагностики |
| NVIDIA Nsight Graphics | NVIDIA graphics/trace/shader analysis и MCP | Для NVIDIA QA |
| WinDbg | CPU crash dumps, стеки, модули | Для диагностики runtime |
| PresentMon | Непрерывная запись frame times | Позже, когда есть осмысленный performance run |

LLVM, Python, CMake, Ninja, RGA, Ghidra и Linux/Mesa полезны для отдельных задач, но **не являются обязательными зависимостями базовой Cargo-сборки** этого SHA. CUDA Toolkit не требуется для D3D12. `Nsight Graphics` нужен именно для графики; установка одного `Nsight Compute` не заменяет его.

В PowerShell установить отсутствующие CLI:

```powershell
winget install --id Git.Git --exact
winget install --id GitHub.cli --exact
winget install --id Microsoft.PowerShell --exact
winget install --id Rustlang.Rustup --exact
```

Установить [Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/) через официальный установщик. Выбрать **Desktop development with C++**, MSVC x64/x86, Windows SDK. Уже установленный совместимый Visual Studio с этим workload подходит. Важно наличие компонентов, а не само название IDE.

После установки открыть новый PowerShell. Для ручной диагностики сборки удобно использовать **Developer PowerShell / x64 Native Tools Command Prompt**, чтобы MSVC и SDK были в окружении.

```powershell
git --version
gh --version
rustup --version
$PSVersionTable.PSVersion
```

Rust `1.98.1`, target `x86_64-pc-windows-msvc`, `rustfmt` и `clippy` заданы в [rust-toolchain.toml](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/rust-toolchain.toml). Не заменять pin на произвольный `stable` ради устранения ошибки установки. В другом выбранном SHA источником версии должен быть его toolchain-файл.

DXC скачать из [официальных Microsoft releases](https://github.com/microsoft/DirectXShaderCompiler/releases), распаковать полный x64 пакет, например в `C:\tools\dxc`. Сохранить вместе `dxc.exe`, `dxcompiler.dll`, `dxil.dll` и остальные файлы пакета. PC DXC не является Xbox compiler.

```powershell
$DxcBin = 'C:\tools\dxc\bin\x64'
if (-not (Test-Path -LiteralPath "$DxcBin\dxil.dll")) { throw 'Не найден dxil.dll x64' }
& "$DxcBin\dxc.exe" --version
$env:VICTORY_DXIL = "$DxcBin\dxil.dll"
$env:VICTORY_PC_DXC = "$DxcBin\dxc.exe"
```

Эти переменные действуют в текущей сессии и её дочерних процессах. Для агента, запущенного отдельно из Desktop, нужно отдельно передать эти пути через его окружение либо launcher. Секретов в этих двух переменных нет.

Windows Graphics Tools включить через **Параметры → Дополнительные компоненты**. Установка может потребовать администратора; обычная работа агентов и сборки должна идти из обычной пользовательской сессии. Наличие компонента проверить фактическим созданием D3D12 debug interface в GPU тесте.

Источники: [VIctory README](https://github.com/Marvisak/VIctory), [официальная настройка Rust на Windows](https://learn.microsoft.com/en-us/windows/dev-environment/rust/setup), [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/install/install-powershell-on-windows), [GitHub CLI](https://github.com/cli/cli#installation).

## 2. Подготовить рабочие каталоги

Ниже предложенные пути. Если на диске C мало места, выбрать другой локальный SSD и последовательно заменить корень.

| Путь | Содержимое |
|---|---|
| `C:\dev\gta6\repo` | Основной checkout форка |
| `C:\dev\gta6\worktrees` | Отдельные checkout задач |
| `C:\gta6-lab\game` | Локальные исходные файлы игры, если доступны |
| `C:\gta6-lab\runs` | Логи, captures, manifests по кандидатам |
| `C:\gta6-lab\cache` | Производные данные по SHA/adapter/режиму |
| `C:\gta6-lab\data` | Отдельные сохранения тестовых запусков |
| `C:\tools` | DXC и независимые инструменты |

```powershell
New-Item -ItemType Directory -Force -Path `
  'C:\dev\gta6', 'C:\dev\gta6\worktrees', `
  'C:\gta6-lab\runs', 'C:\gta6-lab\cache', 'C:\gta6-lab\data' | Out-Null
```

Код и текстовые синтетические fixtures идут в Git; игровые файлы, производные shader dumps, captures и сохранения остаются вне репозитория и всех worktree. Это также соответствует [asset policy VIctory](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/docs/asset-policy.md). Не переносить полный игровой каталог в агентный контекст.

Если используешь существующий каталог с незакоммиченными изменениями, сначала разобрать `git status --short`: зафиксировать нужный исходный код отдельным коммитом, локальные заметки оставить снаружи или добавить точные пути в локальный exclude. Новый worktree строится из **коммита**, он не получает незакоммиченные и untracked файлы исходного checkout.

```powershell
git status --short
git rev-parse --git-path info/exclude
```

`info/exclude` позволяет исключать личные файлы без изменения общего `.gitignore`; это не способ хранения игровых ресурсов внутри repo и не защита секрета от чтения агентом.

## 3. Создать форк и зафиксировать исходную точку

Сначала авторизовать GitHub CLI:

```powershell
gh auth login
gh auth setup-git
$GitHubUser = (gh api user --jq .login).Trim()
$ForkName = 'victory-stability'
$UpstreamRepo = 'Marvisak/VIctory'
```

Создать форк, затем clone в явный путь:

```powershell
gh repo fork $UpstreamRepo --fork-name $ForkName --clone=false
git clone "https://github.com/$GitHubUser/$ForkName.git" 'C:\dev\gta6\repo'
Set-Location 'C:\dev\gta6\repo'
git remote add upstream 'https://github.com/Marvisak/VIctory.git'
git fetch upstream
git remote -v
gh repo set-default "$GitHubUser/$ForkName"
```

Если форк или каталог уже существуют, использовать их после проверки remote и working tree; не запускать создание повторно и не удалять старый каталог. Здесь применяется обычный `git clone`, поэтому `upstream` добавляется явно. При `gh repo clone` он может создаваться автоматически.

Выбрать воспроизводимый baseline:

```powershell
$UpstreamSha = '940d32f69b22265690a0b5a15d6ded572ba33baf'
git show --no-patch --format=fuller $UpstreamSha
git switch -c integration $UpstreamSha
git tag baseline/upstream-2026-11-25 $UpstreamSha
git rev-parse HEAD
```

Если Git не находит этот SHA, сначала проверить `git fetch upstream` и адрес upstream. Не подменять его молча текущим HEAD. Можно осознанно выбрать более новый SHA, повторить аудит и сохранить его в manifest.

Назначение веток:

- `integration`: проверяемые небольшие изменения.
- `task/<id>`: ровно одна задача одного исполнителя.
- `stable`: появится после определённого acceptance suite, а не сразу после форка.

По умолчанию GitHub fork публичного проекта сохраняет публичность исходного репозитория; секреты и локальные данные в push не включать. Существующую лицензию и notices сохранять. Код из других проектов переносить после проверки условий конкретного материала.

Источник команд: [gh repo fork](https://cli.github.com/manual/gh_repo_fork), [Git worktree](https://git-scm.com/docs/git-worktree).

## 4. Проверить инструкции и собрать неизменённый baseline

До первой сборки просмотреть корневые и вложенные инструкции, Cargo manifests, `build.rs`, `.cargo/config.toml`, hooks и CI. Это небольшой обзор того, что сборка будет выполнять, а не полный аудит безопасности.

```powershell
Get-Content .\rust-toolchain.toml
Get-Content .\Cargo.toml
Get-Content .\.cargo\config.toml
Get-Content .\.github\workflows\ci.yml
rustup show
cargo --version
rustc -Vv
```

После прочтения hooks установить предусмотренную проектом проверку данных:

```powershell
cargo run --locked -p xtask -- install-hooks
cargo run --locked -p xtask -- policy
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo build --release --locked -p victory
cargo build --profile diag --locked -p victory --features diagnostics
```

`cargo xtask install-hooks` / `cargo xtask policy` — документированные короткие aliases; явный вызов выше добавляет `--locked` для Cargo. Не применять mass-format или dependency upgrade для того, чтобы скрыть ошибку базового snapshot. Ошибку нужно записать и исправить отдельным обоснованным коммитом.

Все команды выполнять по очереди и проверять exit code. Полный baseline тестов может включать обращения к GPU: проводить его без параллельного игрового процесса и capture.

```powershell
$BaselineDir = 'C:\gta6-lab\runs\baseline-940d32f'
New-Item -ItemType Directory -Force -Path $BaselineDir | Out-Null
cargo test --workspace --locked -- --test-threads=1 --nocapture 2>&1 |
  Tee-Object -FilePath "$BaselineDir\cargo-test.log"
$TestExitCode = $LASTEXITCODE
if ($TestExitCode -ne 0) { throw "Baseline tests failed: $TestExitCode" }
```

Затем один прицельный shader-test проход, если нужно отделить validator/hardware ошибки:

```powershell
cargo test -p victory-shader --lib --locked -- --test-threads=1 --nocapture
```

**Проверить сообщения `skipped` и early-return случаи даже при exit code 0.** Некоторым tests нужен консольный DXC, другим — определённые hardware features. Отсутствие prerequisite — `BLOCKED` или `UNSUPPORTED`, а не доказанный pass. Не искать закрытый SDK ради старта: доступны синтетические tests без него.

Команды и профили сверены с [CI](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/.github/workflows/ci.yml), [CLI](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/apps/victory/src/main.rs), [diff_tests.rs](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/crates/victory-shader/src/emit/diff_tests.rs).

## 5. Настроить пути и выполнить офлайн-инвентаризацию

Если локального пакета игры пока нет, пропустить анализ его файлов и продолжить синтетические tests. Окружение и первые задачи можно готовить без него.

```powershell
if (-not (Test-Path -LiteralPath '.\victory.toml')) {
  Copy-Item .\victory.example.toml .\victory.toml
}
notepad .\victory.toml
```

Пример содержимого локального файла:

```toml
[paths]
game_dir = 'C:\gta6-lab\game'
cache_dir = 'C:\gta6-lab\cache\baseline-940d32f'
data_dir = 'C:\gta6-lab\data\baseline-940d32f'
test_titles = []
```

Переменные `VICTORY_GAME_DIR`, `VICTORY_CACHE_DIR`, `VICTORY_DATA_DIR` имеют приоритет над файлом; `VICTORY_CONFIG` выбирает иной config. Перед baseline проверить только переменные `VICTORY_*`, чтобы старые экспериментальные switches не изменили запуск. Не публиковать общий dump всех environment variables с credentials.

```powershell
Get-ChildItem Env:VICTORY_*
cargo run --locked -p victory -- config
```

Проверить не только код выхода, но и выведенные пути/`POLICY VIOLATION`. Выбрать реальный title EXE из своего пакета:

```powershell
$GameDir = 'C:\gta6-lab\game'
$TitleExe = Read-Host 'Полный путь к основному EXE игры'
if (-not (Test-Path -LiteralPath $TitleExe -PathType Leaf)) { throw 'EXE не найден' }
$ShaderPack = Join-Path $GameDir 'shaders.rpf'
if (-not (Test-Path -LiteralPath $ShaderPack -PathType Leaf)) { throw 'shaders.rpf не найден' }
$VictoryDiag = '.\target\diag\victory.exe'

& $VictoryDiag inspect $TitleExe --imports
& $VictoryDiag census $GameDir --missing
& $VictoryDiag shader-info $ShaderPack --verbose
& $VictoryDiag translate $ShaderPack --validate --list --jobs 4
```

Сохранить stdout/stderr каждого анализа в каталог baseline. `translate` собирает отказы и может вернуть 0: нужен просмотр числа отказов и причин, позднее — структурированный отчёт из `MS001`.

`--pipelines` дополнительно создаёт PSO на GPU: выполнять только в выделенной GPU сессии после проверки, какой адаптер реально выбран. До `MS002` не утверждать, что тест шёл на iGPU.

Прогрев — **отдельная** операция, когда он нужен для соответствующего кандидата:

```powershell
$ShaderInputs = @($ShaderPack, $TitleExe,
  (Join-Path $GameDir 'XboxBvhBuilder.dll'),
  (Join-Path $GameDir 'amd_fidelityfx_upscaler_xbox.dll'))
foreach ($InputPath in $ShaderInputs) {
  if (-not (Test-Path -LiteralPath $InputPath -PathType Leaf)) {
    throw "Не найден вход: $InputPath"
  }
}
& .\target\release\victory.exe translate --cache --pipelines --jobs 4 @ShaderInputs
```

Нельзя объединять `--cache` с `--validate`/`--list`. Прогрев не доказывает корректность и не гарантирует отсутствие refused. Pipeline cache зависит от адаптера/драйвера; используйте отдельные writable каталоги для кандидатов и режимов.

Источники: [конфигурация](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/crates/victory-core/src/config.rs), [translate](https://github.com/Marvisak/VIctory/blob/940d32f69b22265690a0b5a15d6ded572ba33baf/apps/victory/src/translate.rs).

## 6. Подготовить два GPU без дополнительной ОС

В BIOS оставить встроенную графику активной одновременно с PCIe GPU. Точные названия пунктов зависят от версии BIOS MSI; универсального пути меню здесь нет. RTX может оставаться основным выводом изображения. Установить официальный драйвер AMD для встроенной графики.

```powershell
Get-CimInstance Win32_VideoController |
  Select-Object Name, PNPDeviceID, DriverVersion
```

Это проверка видимости устройств, **не проверка D3D12 capabilities**. `MS002` должен получить DXGI LUID, VendorID/DeviceID, software flag, shader model, WaveOps/min/max lanes, mesh/RT capabilities и явно выбрать адаптер. В текущем test host эвристика `DedicatedVideoMemory > 0` может неправильно исключать UMA; готового проверенного `VICTORY_ADAPTER=amd` нет.

Небольшие D3D12 compute tests на AMD и NVIDIA не требуют cross-adapter shared memory: каждый GPU получает свои входные буферы, результаты после fence сравниваются на CPU. Подробности и границы эталона — [REFERENCE_TEST_PLAN_RU.md](REFERENCE_TEST_PLAN_RU.md).

## 7. Настроить Hermes

Использовать уже работающую native Windows установку либо [официальный установщик](https://hermes-agent.nousresearch.com/docs/getting-started/installation). Если Hermes пока живёт на Mac, сервере или в WSL, обеспечить отдельный Windows исполнитель с доступом к GPU; для кратчайшего старта проще локальный Hermes на этом ПК.

```powershell
hermes --version
hermes setup
hermes model
hermes profile list
```

Не перестраивать рабочую авторизацию, если она уже настроена. Авторизация модели и авторизация GitHub MCP — разные вещи. Не подменять официальную авторизацию переносом неизвестной структуры `auth.json`.

Создать `gta6-lead`, `gta6-impl`, `gta6-review`, `gta6-gpu` согласно [HERMES_SETUP_RU.md](HERMES_SETUP_RU.md). Для небольшого стартового объёма достаточно одного implementer; второй нужен, когда появились независимые задачи. Модель выбирать из реально доступных в установленном клиенте, без обещания измеренных множителей ускорения.

Lead ведёт план и интеграцию, implementer меняет назначенный worktree, reviewer читает точный candidate SHA, GPU QA выполняет аппаратную проверку. Одна GPU сессия в один момент времени.

## 8. Подключить MCP

Подробные готовые настройки — [MCP_SETUP_RU.md](MCP_SETUP_RU.md) и `templates`.

1. **Microsoft Learn**: документация D3D12/DXGI/Windows, без auth.
2. **GitHub MCP**: опционально, чтение source/issues/PR; для Git уже есть `gh`.
3. **Nsight Graphics MCP**: NVIDIA graphics captures и trace analysis, Windows preview.
4. **PIX Timing Captures MCP**: исследование `.wpix`, CPU/GPU events, I/O и памяти.

Learn/GitHub доступны lead и reviewer; локальные GPU MCP на старте получает только GPU QA. Не требуется подключать десяток дублирующих filesystem/git/memory серверов. Самописный MCP имеет смысл после появления проверенного CLI runner.

После подключения провести три проверки: `tools/list` успешно; локальный сервер видит synthetic/sample capture по абсолютному пути; агент правильно описывает один подтверждаемый факт из него. До MS003 использовать собственное минимальное приложение или официальный sample, не запуск игры. Одной строки «server configured» недостаточно.

## 9. Зафиксировать инструкции и создать worktree

Сначала согласовать `templates/AGENTS.project-template.md` с действующим upstream `AGENTS.md`, `.hermes.md`/`HERMES.md` и вложенными инструкциями. Не сохранять одновременно противоречащие наборы. Hermes может выбрать более приоритетный файл и не загрузить `AGENTS.md`.

Создать в форке `docs/agent-setup/` и `docs/tasks/`, перенести выбранные инструкции/задачи. Конфиги с токенами и абсолютные пути пользователя оставить вне tracked файлов. Для первых двух задач достаточно MS000/MS001.

```powershell
git status --short
git add AGENTS.md docs/agent-setup docs/tasks
git diff --cached --stat
git diff --cached
git commit -m "docs: define reproducible compatibility development workflow"
git push -u origin integration
$BootstrapSha = (git rev-parse HEAD).Trim()
git worktree add -b task/MS000 'C:\dev\gta6\worktrees\MS000' $BootstrapSha
git -C 'C:\dev\gta6\worktrees\MS000' rev-parse HEAD
git -C 'C:\dev\gta6\worktrees\MS000' status --short
git worktree list
```

Перед `git add` должны существовать именно эти выбранные файлы и каталоги. Если у тебя другой layout, перечислить его явно. Не делать `git add .` в каталоге с локальными экспериментами.

В worktree проверяются ожидаемый SHA и наличие `AGENTS.md`/спецификации задачи. Если их нет, это проблема исходного коммита, а не повод расширять разрешения агента на соседние каталоги. Игнорируемый `victory.toml` не переносится Git: настроить его в каждом worktree либо использовать явный `VICTORY_CONFIG` на файл вне repo.

## 10. Выдать первую задачу и проверить маршрутизацию

Выполнить документированные команды Kanban из `HERMES_SETUP_RU.md`: board, diagnostics, create с явными `assignee` и `dir:<существующий worktree>`, `dispatch --dry-run`, затем один worker.

Перед интерактивной lead-сессией явно выбрать доску через `hermes -p gta6-lead kanban boards switch gta6` и проверить `boards show`. Удалить или согласовать старое значение `HERMES_KANBAN_BOARD`, если оно указывает на другой проект: environment имеет приоритет над current pointer. Текст «Доска gta6» в промпте не заменяет фактическую маршрутизацию. Сохранить task ID уже созданного MS000; lead должен продолжить эту карточку, а не создавать дубликат.

В тело первой карточки включить путь к `docs/tasks/MS000.md`, base SHA, ожидаемые артефакты и запрет на изменения семантики. Первую карточку не завершать по одному факту «скомпилировалось»: нужен перечень реально выполненных и заблокированных проверок.

Рекомендуемая последовательность разработки:

| Задача | Результат |
|---|---|
| MS000 | Baseline, inventory, реальные PASS/FAIL/BLOCKED, версии и источник результатов |
| MS001 | Структурированный отчёт offline translator и строгий aggregate exit policy |
| MS002 | Выбор AMD/NVIDIA по проверенной идентичности; capability report; тесты не уходят на другой GPU |
| MS003 | Runtime не выдаёт неподдерживаемый draw/dispatch за выполненный; диагностируемая остановка |
| MS004 | Один независимый synthetic ISA regression; минимальное исправление при подтверждённом дефекте |

MS001/MS002 можно готовить независимо после согласования schema. Все аппаратные проверки выполняются последовательно. MS003 — изменение runtime-политики, его собственный тест должен намеренно подавать неподдерживаемую операцию и получать ожидаемый диагностический отказ. Это **положительный тест обработки ошибки**, а не успешный рендер.

## 11. Определить передачу кандидатов и приёмку

Исполнитель передаёт exact commit SHA, base SHA, изменённые файлы, rationale, commands и результаты. Reviewer проверяет этот commit, а GPU QA запускает собранный из него binary с известным hash. Нельзя тестировать старый executable из соседней ветки.

После `CHANGES` исправляется тот же scope; новая ревизия снова получает SHA и проверку. После интеграции повторяется затронутый минимальный набор. Длительные прогоны увеличиваются только для конкретного риска: например, leak, streaming или редкая гонка. Зелёный CI без аппаратуры не объявляется GPU validation.

Для correctness run включать Debug Layer и нужную GBV; DRED должен быть включён до создания устройства. Для performance run использовать тот же код/сцену/проверенную семантику, но убрать тяжёлую instrumentation и зафиксировать режим. GBV меняет шейдеры и GPU workload, поэтому её timings не являются production timings. [Microsoft GBV](https://learn.microsoft.com/en-us/windows/win32/direct3d12/using-d3d12-debug-layer-gpu-based-validation), [DRED](https://learn.microsoft.com/en-us/windows/win32/direct3d12/use-dred).

В performance report нужны frametime distribution, stalls, adapter, driver, scene/build, cache state, log всех deviations. 30 FPS — бюджет 33,33 мс между кадрами; CPU и GPU могут перекрываться, поэтому прежняя простая сумма бюджетов не является моделью pipeline. Достаточность железа и срок достижения цели пока не измерены.

## 12. Критерий готовности к разработке

Начинать MS001/MS002 можно, когда:

- upstream SHA и собственный bootstrap SHA записаны;
- компиляция и состояние baseline tests воспроизводятся;
- существующие пропуски/отказы видны и не маскируются;
- рабочее дерево задачи содержит согласованные инструкции;
- нужный профиль действительно назначается исполнителем;
- доступ к файлам/CLI/MCP проверен конкретной операцией;
- есть один владелец GPU очереди и отдельные writable cache/data;
- готов маленький synthetic test scope, не требующий консольного SDK.

Полный реверс Xbox, готовый интерпретатор всех GPU инструкций и покупка ещё одной видеокарты не являются условиями начала. Первая полезная инженерная цель — достоверный результат одного проверяемого преобразования и ясная ошибка для того, что ещё не реализовано.
