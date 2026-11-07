# Workspace instructions

<!-- monorepos:begin -->
## Monorepos multi-projets

- Un monorepo possède un seul dépôt Git à sa racine. Ses modules et compagnons de tests sont des projets internes, sans `.git` enfant ni sous-module. Ne pas convertir un autre écosystème sans demande explicite.
- Les dépendances de modules restent déclarées dans leurs manifestes ; un seul dépôt ne signifie pas un seul paquet publié. Conserver les tests dans leurs projets compagnons.
- Définir la CI par chemins et par dépendances : un changement d’un socle valide aussi ses consommateurs ; un changement d’un consommateur peut lancer uniquement ses suites. Les scripts communs, le SDK et le protocole déclenchent les vérifications transversales.
- Utiliser un workflow réutilisable pour les étapes communes. Les workflows filtrés par `paths` l’appellent avec le périmètre attendu ; ne pas dupliquer les commandes de build/test dans chaque filtre.
- Un filtre de chemins n’est pas une preuve de qualification. Une publication reconstruit et valide tous les paquets, leurs dépendances et leurs plateformes attendues, même si la dernière CI de développement était limitée à un projet.
- Les commits et pushes portent sur le monorepo. Ne pas créer de commit indépendant dans un sous-projet ; ne pas committer ni pousser implicitement pendant un build ou un test.
- Documenter au `AGENTS.md` de chaque monorepo la carte des projets, les dépendances, les filtres de workflows, les commandes de validation et le déclenchement de publication. QuietCli2 applique ces règles dans [ses instructions](PoSh/FrenchExDev/QuietCliEcosystem/FrenchExDev2.QuietCli.PoSh/AGENTS.md).
<!-- monorepos:end -->

## PowerShell process startup

- Use `pwsh` with its normal profile by default, including background runners and monitoring windows. With Codex `exec_command`, set `login: true`; do not add `-NoProfile`.
- Reuse the profile's UTF-8 initialization and DevPoSh module autoload. Invoke the required commands directly instead of repeating encoding setup or `Import-Module` boilerplate.
- Import explicitly only when autoload is unavailable or the task requires a particular source version. Use `-NoProfile` only for tasks requiring isolation; initialize UTF-8 and missing modules in that case.

<!-- quietcli:begin -->
## QuietCli2 local engine

Every agent native CLI invocation uses `quiet`. Use the `quiet` alias exported by the local Core V2 module for all native CLI calls. Define the question, business result and smallest useful delivery before executing; read installed help or official documentation before writing executable-specific analysis.

Use PowerShell Core 7.6+ with its normal profile (`login: true` with Codex `exec_command`). Reuse its UTF-8 initialization. The selected source root is `C:/code/FrenchExDev/FrenchExDev_i2/PoSh/FrenchExDev/QuietCliEcosystem/FrenchExDev2.QuietCli.PoSh`. Read its `AGENTS.md`, `README.md` and `FrenchExDev2.QuietCli.Core.PoSh/README.md` before use.

- In each new PowerShell process, use the selected local Core module. If that exact module is not loaded, import `<source-root>/FrenchExDev2.QuietCli.Core.PoSh/FrenchExDev2.QuietCli.Core.PoSh.psd1` explicitly once. For Git, also import `<source-root>/FrenchExDev2.QuietCli.Git.PoSh/FrenchExDev2.QuietCli.Git.PoSh.psd1`. Check the loaded module path; do not silently select an older installed release. Do not load the legacy QuietCli or archived OPumper modules into that process.
- Existing verified binaries under `lib/` are required. Import does not compile C#. No SDK or rebuild is needed to use a matching built module. Rebuild missing or stale binaries with the monorepo's documented `scripts/Build-Workspace.ps1`, then start a fresh PowerShell process. Do not rebuild or install dependencies automatically on every command.
- Leave hosting at `Auto`: CLR 10 uses the integrated net10.0 build; CLR 11+ uses the compatible net11.0 build. Never load the .NET 11 assembly into CLR 10. Explicit `Process` hosting requires the pinned .NET 11 runtime. `Get-QuietEngine` reports the actual selected host.
- Example: `$r = quiet -AsObject -C work -WorkingDirectory $repository git status`. Facade options precede the executable; native arguments follow it verbatim. For exact argument vectors, use `-FilePath git -ArgumentList @('status','--','a file')`.
- Use a documented registered handler or direct `-Output { param($event,$runtime) ... }` / `-Complete { param($runtime) ... }` callbacks. Without either, an unsupported command is rejected before business execution. Callbacks emit no incidental pipeline output; retain only the required business data in `runtime.Data`.
- Keep `-AsObject` results in variables. Inspect `Outcome`, `Execution`, `Completeness`, errors and business scope. Data stays local and is not enumerated or serialized by `-AsObject`. Deliver only a bounded view through `FrenchExDev2.QuietCli.Core.PoSh\Out-QuietResult` or `FrenchExDev2.QuietCli.Core.PoSh\ConvertTo-QuietJson`.
- Native output is streamed once with bounded buffers; no automatic raw capture, spool or content history is created. SQLite stores metadata and counters, not business output. QuietCli2 has no RECORD/replay. Do not use legacy `-ContractPath`, `-OnOutput`, `-Record`, `Get-QuietRecord`, `Invoke-QuietRecord`, `Test-QuietModule`, `Install-QuietModule` or `Get-QuietAudit` APIs as V2 instructions.
- Reusable executable semantics belong in a companion business module with independently asserted tests. Inventory membership is not qualification; keep unsupported variants and version/platform gaps explicit.
- Inspect `Get-QuietCliGain` for measured volumes and overhead. Unknown native token counts and unmeasured delivery sizes remain unknown; `-AsObject` alone does not establish a serialization gain.
- No automatic retry, hook installation, network access, profile change, publication or escalation is authorized by the engine. CI/build orchestration retains the monorepo's engine-independent `scripts/Ci.Process.ps1` helper. Test execution retains the visible monitor rules; read-only streaming monitors remain the native-wrapper exception.

<!-- quietcli:end -->

<!-- tests:begin -->
## Test runs

Every test run started by an agent also gets a visible PowerShell Core (`pwsh`) monitoring window. Start the tests in the background and open that window at the same time, for the whole run. A monitor confined to tool output or chat does not satisfy this requirement. The Claude Code `Monitor` tool may supplement the window.

- Open the dedicated monitoring window explicitly (for example, `Start-Process pwsh -WindowStyle Normal -ArgumentList ...`). Give the user the command or script path to reopen it.
- These monitoring rules apply to test runs and mutation campaigns. Automatically close each dedicated monitoring window once all runs it tracks have reached a terminal state (success, test failures, crash, timeout or cancellation), after saving the final summary and diagnostics to the run's output files. Do not ask the user whether to close the window, wait for keyboard input or use `-NoExit`. End the monitoring process normally; keep the saved results available for later consultation or reopening.
- When Windows and Linux runs execute together, show both in the monitoring window, with their actual process or container identity, elapsed time and independent results.
- Render the monitoring summary as a bordered text table with literal `|` separators between every column, aligned column widths and one line per run. Include platform, state, elapsed seconds, process/container identity and available test counters. Show `-` for unknown counters; keep the table narrow enough to avoid line wrapping. Plain `Format-Table` output without `|` separators does not satisfy this requirement.

- The monitor reports progress and every terminal state: success, test failures, crash and timeout. Silence must never be the only sign of an outcome. Re-arm it if it expires before the run ends.
- When the runner emits no per-test progress, report a periodic heartbeat from observable signals such as elapsed time, output lines and temporary directories, and do not present these as a test count.
- The monitor is read-only and only reads the run's own output and progress files. This is the one exception to running CLI commands through `quiet`, because a monitor must stream its events; the tests themselves keep following the rules above.
<!-- tests:begin -->

<!-- tables:begin -->
## UTF-8 et tableaux

- Toujours forcer explicitement UTF-8 pour les fichiers texte, scripts, lectures, écritures et échanges avec les commandes, ainsi que pour les entrées et sorties du terminal. Préserver les accents et les caractères Unicode.
- En PowerShell, réutiliser l'initialisation UTF-8 du profil sans répéter le préambule. Si le profil est absent ou désactivé, initialiser l'encodage avant toute entrée/sortie :

```powershell
$utf8 = [System.Text.UTF8Encoding]::new($false)
[Console]::InputEncoding = $utf8
[Console]::OutputEncoding = $utf8
$OutputEncoding = $utf8
```

- Lorsqu'un tableau est demandé, utiliser par défaut les caractères de cadre Unicode encodés en UTF-8 : `┌ ┬ ┐ ├ ┼ ┤ └ ┴ ┘ ─ │`. Cette préférence couvre les tableaux textuels, les consoles et les fenêtres de suivi, y compris lorsque le mot « ASCII » est employé, sauf demande explicite d'ASCII strict.
- Aligner les colonnes et les bordures, adapter leur largeur au terminal et aligner les valeurs numériques à droite.
- Dans toute interface terminal persistante ou fenêtre de suivi, les tableaux doivent être dynamiques au redimensionnement : détecter les changements de dimensions du terminal, recalculer les largeurs des colonnes et redessiner le tableau, même si les données n'ont pas changé. Une largeur calculée uniquement au démarrage ne suffit pas.
- Mesurer la largeur réellement disponible et la largeur d'affichage des caractères Unicode ; ne pas compter les séquences ANSI comme des caractères visibles. Adapter les colonnes, les bordures et les cellules sans provoquer de retour à la ligne automatique ni désaligner le cadre. Si nécessaire, tronquer explicitement avec `…`, masquer les colonnes secondaires ou utiliser une présentation verticale lorsque le terminal devient trop étroit.
- Rafraîchir uniquement la zone de l'interface avec un cadrage stable, sans `Clear-Host` ni clignotement. Conserver les informations d'identification du projet et de la tâche visibles après chaque redimensionnement ; adapter également le nombre de lignes visibles à la hauteur disponible.
- Pour une sortie ponctuelle, calculer la disposition à partir de la largeur courante au moment de l'affichage. En sortie redirigée ou sans terminal interactif, produire un tableau statique lisible avec une largeur de repli explicite, sans séquences de repositionnement du curseur ni boucle de surveillance du terminal.
<!-- tables:end -->

<!-- Async:begin -->
## Asynchronous programming

Always prefer async methods over sync ones.
Even in synchronous powershell, use .GetAwaiter().GetResult();
<!-- Async:end -->

<!-- Comments:begin -->
## Documentation

- Write documentation for functions in Powershell, for classes, interfaces, methods, parameters for C#.
- Always write in english

<!-- Comments:end -->

<!-- Terminal:start -->
## Terminal UI

Always use console framing instead of Clear-Host, to avoid blinking

- The first content line of every terminal UI and monitoring window must identify the project being worked on and the current task. Example: `Projet : AIWebsiteDetector | Tâche : mutation C# et PowerShell`.
- Identify the concrete project or module selected by the tests or mutation campaign, such as `FrenchExDev.DevPoSh.PoSh`. An enclosing repository or workspace name such as `FrenchExDev_i2` is insufficient when it contains several projects. Show the full project directory as well as the workspace directory, and name each target when multiple projects share a monitor.
- Display the absolute workspace path immediately below that first line, and use the project name and task in the window or tab title.
- For a shared monitor, identify each project or module being tested in the heading or its own row. Keep this identity visible during refreshes and in the saved final summary.
- For durations in any table, use the column header `Duration` and compact units in cells: `1s`, `1m`, `1h`, `1m30s`, `2h5m`. Apply this to terminal and monitoring tables, and display `-` when the duration is unknown.
- Make cursor disabled when using/coding watchers
<!-- Terminal:end -->

<!-- Testing:start -->
## Testing

Always use existing tests, if one does not exist, create it in test directory; reuse it later
<!-- Terminal:end -->

<!-- Testing:start -->
## Language

Always write code and comments in english. You can talk in UI in user language.
<!-- Terminal:end -->

<!-- Agents.md:start -->
## AGENTS.md

Always write code and comments in english. You can talk in UI in user language.
<!-- Agents.md:end -->

<!-- test-strategy:begin -->
## Property testing, fuzzing and mutation testing

- Property-based testing, fuzzing and mutation testing are mandatory parts of the test strategy for every implementation language, including PowerShell, C#, C and future languages. Consider all three when writing or changing tests; ordinary example tests alone do not satisfy this strategy. Reuse and extend existing companion test projects, generators, corpora, mutation catalogues and runners.
- Define independent business properties and invariants, generate varied valid inputs, and fuzz malformed inputs, boundary values and relevant state/stream sequences. Use mutation testing to verify that assertions detect meaningful behavior changes; assertions must not merely duplicate the implementation. Keep each language's coverage and remaining gaps explicit.
- Execute the relevant campaigns when necessary: new or changed behavior, parsers or serializers, protocols or input validation, state/lifecycle/concurrency logic, a bug exposing a missing invariant, changes to assertions/generators/corpora/mutation tooling, or a dependency/runtime change affecting those guarantees. Run a deterministic regression for each discovered defect and retain its minimized reproducer or seed.
- Scope runs to the affected component and its dependent behavior. Do not rerun every campaign after every edit, read-only operation or documentation-only change. Reuse successful evidence only while its source, tests, dependencies, runtime and configuration remain applicable; explain the selected scope and why existing evidence is sufficient when no new campaign is needed.
- Before release/publication, execute the full required qualification matrix for the delivered revision. A targeted development run does not replace release qualification. Missing tooling or an unexecuted required campaign is an explicit qualification gap, never an implicit pass or an automatic waiver.
- Bound campaign duration and resources, retain seeds/corpora and source identity, and report property/fuzz executions and mutation outcomes separately. Investigate surviving mutants; distinguish equivalent mutants with evidence from missing assertions, invalid mutants, crashes, timeouts and infrastructure failures. Preserve failures and respect the repository's visible-monitor and artifact rules.
<!-- test-strategy:end -->

<!-- design-principles:begin -->
## SOLID, DRY and modular design

- Classes and modules must follow SOLID, DRY and MODULAR design principles in every implementation language. Apply these principles when creating, changing or reviewing code.
- Give each class/module a cohesive responsibility and a name describing that responsibility. Separate parsing, business policy, orchestration and infrastructure; keep dependencies explicit and interfaces focused. Depend on stable contracts, preserve substitutability and extend behavior without scattering command-specific branches across unrelated components.
- Reuse shared behavior through small composable components; avoid duplicated business rules, validation or protocol logic. Keep genuinely different semantics separate instead of forcing unrelated behavior into one abstraction.
- Preserve module boundaries and dependency direction. Do not introduce speculative frameworks, catch-all utility classes, unnecessary interfaces or extra IPC to satisfy a pattern mechanically. Test observable contracts and invariants independently of implementation details.
<!-- design-principles:end -->

## Text encoding and line endings

All text files must use UTF-8 and LF line endings on disk. Honor repository .gitattributes and .editorconfig rules, and generate text/JSON with LF. Preserve binary resources and recorded native evidence; refresh affected content/provenance hashes when normalizing locked text.
