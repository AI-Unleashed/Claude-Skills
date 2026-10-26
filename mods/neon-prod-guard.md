# neon-prod-guard

<!--
Note: some lines in the source copy were cut off mid-word and are reproduced as-is.
Affected spots include: "the hook is skipped and the he inside a mods API call is exempt",
"Anything but Proceed els.", "a diff against the lffs against the live DB", "name, subagen",
"keyed by et() for status", "message from$. });", "If the analysis faable", "stampe hold start",
"neonctlse neonctl on PATH", "the production write, n with another connection string",
"leave it open on cance", "atom/rpes/index.d.ts", "succeeo headline", "$.ui.invalidate ncelled",
"the schema and the app a risk chip", "What changcode affected", "the nfrom",
"config in$.stan be verified", "productionEnvFilesisModel", "snapshot createroject-id",
"the Markdownreport", "dev-mods ettings.json", "type-checkd's declarations", "prodBranchues".
Fill these in from the original before using the prompt.
-->

## GOAL

Any time Claude is about to change my production Neon database, hold the call, show me what will change, have a read-only agent analyse the impact, make me confirm twice (the second time by typing the production branch name), snapshot production to a Neon branch first, and only then let the call run. The mod only adds friction: it must never approve, auto-allow, or answer a tool call with a result of its own. Every exit is either passing the original call through unchanged or refusing it.

## STEP 1: INSPECT THE PROJECT FIRST AND REPORT

Before writing code, tell me:

* How schema changes and migrations are applied (drizzle-kit, Prisma, psql, npm scripts, the Neon MCP server, neonctl).
* Where connection strings live (.env files, Vercel env, etc.) and which env file each tool loads under which NODE_ENV.
* Which Neon endpoint ID (the ep-... host label, without -pooler) is production, and which is dev. If you cannot determine the prod endpoint with certainty, ask me. Don't guess.
* Use neonctl (npx neonctl; I may need to run `npx neonctl auth`) to list the org, project, branches (names and br-... ids) and each branch's endpoint. If the account belongs to more than one organization, neonctl prompts for one; pass --org-id everywhere.

## STEP 2: DETECTION

* Hook tool.call for Bash, and for Neon MCP tools (matcher /^mcp__neon__/) that write. Maintain an explicit set of Neon MCP write tools (run_sql, run_sql_transaction, complete_database_migration, complete_query_tuning, delete_branch, reset_from_parent, update_branch, set_default_branch, finalize_branch_restore, role/database create/update/delete, credentials, delete_project, update_project, endpoint update/delete/suspend/restart, Neon Auth writes, Data API writes, functions and triggers writes, storage deletes/creates, snapshot delete/restore/schedule, explain_sql_statement only when analyze is true). Neon MCP args are snake_case: project_id, branch_id, parent_branch_id (for complete_database_migration), endpoint_id, sql, sql_statements, migration_sql.
* Bash detection must be position-aware: split the command into simple commands (on newlines, ;, &&, ||, |, $( and parentheses), strip heredoc bodies and comments, strip leading VAR=value assignments and wrappers (sudo, env, time, nohup, exec, npx, bunx, pnpm/npm/yarn dlx|exec, dotenv -e FILE --, cross-env VAR=x), open up `bash -c "..."` / `sh -c` strings and `eval`, then match the program at the start: drizzle-kit push|migrate; (npm|pnpm|yarn|bun) [run] db:push|db:migrate; prisma migrate deploy|dev|reset|resolve and prisma db push|execute; psql; pg_restore; neonctl/neon branches delete|reset|restore|rename|set-default|add-compute|remove-compute, sql, projects delete, databases create|delete, roles create|delete|reset-password. The word "psql" inside an echo string or a heredoc body must NOT match (this bit me: a Python heredoc mentioning psql tripped the guard).
* Identify prod by endpoint ID, never by words like "prod" in the command. Resolve the connection string a command will actually use, in this precedence: an explicit URL in the command (--url flag, positional postgres:// URL, DATABASE_URL=literal); DATABASE_URL=$VAR traced back to a VAR=$(grep ... .env.production.local ...) definition in the same command (the env file named there); --env-file / dotenv -e / source FILE; psql -h host or PGHOST; DATABASE_URL already in the session environment (dotenv never overrides it); otherwise the env file the tool loads: the production env file when NODE_ENV=production is set inline, else the default env file (both configurable, defaults .env.production.local and .env.development.local). Honour a leading `cd DIR &&` when resolving file paths. A resolved non-Neon host (localhost) is "not production" and passes.
* Fail closed: if a command writes to a database and the target cannot be resolved (unknown variable, missing file, no DATABASE_URL in it, MCP call with no branch_id, prodBranchId unconfigured, prodEndpointId unconfigured), treat it as production.

## STEP 3: HOLD MECHANICS (important engine constraint)

A hook's own waiting counts against a 10-second limit, after which the hook is skipped and the he inside a mods API call is exempt. So do NOT wait on pane buttons. Use $.ui.ask (the engine's AskUserQuestion dialog) for both confirmations; the pane is for display. Attach a .catch handler to both tool.call hooks that denies when next was not called (fail closed on throw/timeout).

Before holding, check $.session.surfaces(): if neither 'terminal' nor 'desktop' is attached (VS Code panel, claude -p, cloud), refuse production writes with a deny explaining why.

## STEP 4: CONFIRMATION 1

Open a pane (id neon-prod-guard, title "Neon prod guard", not focused) and ask: "Change the PRODUCTION database with <tool>? Target: <branch, endpoint, how it was resolved>. N destructive statements; the pane shows the SQL. Proceed?" with options ['Cancel', 'Proceed'] and header "Prod 1 of 2". Anything but Proceed els.

SQL preview: for drizzle-kit push, copy <out>/meta into node_modules/.cache/neon-prod-guard/preview-<stamp> (the --out path must be relative) and run `npx --no-install drizzle-kit generate --dialect X --schema Y --out <rel>` with stdin closed and a timeout; read the generated .sql; label it as a diff against the lffs against the live DB; if the output contains "created or renamed", say drizzle-kit needs an interactive rename answer and show no SQL; delete the preview folder. For drizzle-kit migrate, show the newest file from the journal. For psql, -c SQL, -f file, < file, or a heredoc body. For MCP, the sql/sql_statements/migration_sql argument. Classify statements (split on ; and `--> statement-breakpoint`): DROP ..., TRUNCATE, ALTER TABLE ... DROP COLUMN/CONSTRAINT, ALTER ... TYPE, DELETE without WHERE, UPDATE without WHERE are destructive.

## STEP 5: IMPACT ANALYSIS BETWEEN THE CONFIRMATIONS

After Proceed, spawn a read-only subagent with $.agent.spawn({ prompt, description, name, subagen (a plugin cannot call the Agent tool directly; retry without subagentType if the type is refused). Give it the target, tool, command, previewed SQL, the working directory, and this brief: investigate read-only (schema file, migrations, grep app code for affected tables/columns), never run anything that writes, reply in markdown under 250 words with first line `RISK: LOW|MEDIUM|HIGH - why`, then sections "What changcode affected", "Before proceeding".

Wait for it without spending the hook budget: record answers from a turn.complete hook keyed by et() for status; between polls call $.process.run(['sleep','2']) (a host call, exempt) falling back to $.clock.sleep; when completed with no recorded answer, read the last assistant message from$. }); cap at about five minutes. If spawn returns no agentId (the test kit does this), find the agent by the name you gave it. Store the report in $.store under lastAnalysis. If the analysis faable: <why>" and continue.

Options: analysis = agent|off (default agent), analysisModel = sonnet|haiku|opus|default.

## STEP 6: CONFIRMATION 2, TYPE THE TARGET

Ask: "<RISK line> (full analysis in the pane). Second confirmation: choose Other and type the production branch name exactly to continue, or cancel?" with options ['Cancel', 'Cancel and use the dev branch instead'] and header "Prod 2 of 2". The call continues only if the typed text (trimmed) equals prodBranchName exactly. "Cancel and use the dev branch instead" cancels with a deny that tells Claude to target the default env file without NODE_ENV=production. If prodBranchName is unconfigured, cancel and say so.

## STEP 7: SNAPSHOT

Before letting the call run, create a branch off prod named pre-push-<YYYYMMDD-HHMMSS UTC, stampe hold start>: `neonctl branches create --project-id <id> --org-id <org> --parent <prodBranchName> --name <name> --no-compute --no-secrets -o json` with stdin closed and a generous timeout. neonctlse neonctl on PATH, else `npx --yes neonctl@8`. Project: the configured projectId, else `neonctl projects list -o json` if it returns exactly one. Parse the br-... id from the JSON. Build the restore command: `<neonctl> branches restore <prodBranchName> <snapshotId> --project-id <id> --org-id <org>`. If the snapshot fails, cancel the push with neonctl's error. Never push without a snapshot.

## STEP 8: OUTCOME

* Cancel: answer the call with { deny } telling Claude the user cancelled the production write, n with another connection string, and to ask the user.
* Success: $.ui.log a transcript line with the snapshot name, id, and "Roll back with: <restore command>"; store lastSnapshot in $.store; then next(e) and return its result unchanged. Mark the pane done or failed by the result. Close the pane by itself 10 seconds after a success (clock.after); leave it open on cance

## PANE DESIGN

(ui.render for component Pane, requestId neon-prod-guard; state in $.state via atom/rpes/index.d.ts)

* Header card: rounded border coloured by state (error while held, warning while analysing, succeeo headline (PRODUCTION WRITE HELD / RAN / CANCELLED / FAILED), a braille spinner with an elapsed-seconds counter while waiting (redraw once a second with $.clock.every + $.ui.invalidate ncelled in a finally), a step tracker "✓ Confirm → ● Analyse → ○ Type name → ○ Snapshot → ○ Run" with done steps green, the current one inverse, and the step's status sentence.
* Target card: inverse red PRODUCTION badge (or "TARGET UNKNOWN: TREATED AS PRODUCTION"), branch name, endpoint in the claude theme colour, "resolved from ...", tool, and the command as a Code block (bash, or json for MCP).
* SQL card: border red when anything is destructive, a badge "N statements, D destructive", the nfrom, each destructive statement called out in red ("!! DROP COLUMN: ..."), and the SQL in a Code block with language sql.
* Analysis card: while running, an animated "A read-only agent is reading the schema and the app a risk chip (green/yellow/red by level) and the report as Markdown.
* Snapshot card: green border, name and id, the rollback command as a bash Code block.
* Buttons: "copy rollback" (hotkey c, $.ui.copy, toast the result) and "close" (hotkey x).

## COMMANDS (/neon-guard)

* no args: config summary. check: resolve neonctl, the project, list branches, confirm prodBranchName exists and report its br- id. set <option> <value>: $.config.set on neon-prod-guard.<option>. resolve <command>: explain what the guard would do with a command without running it. close.
* At session.start, $.ui.log one line saying what is guarded, and store the loaded config in$.stan be verified from outside.

## CONFIG

(userConfig, none required so the mod never fails to load; unconfigured means fail closed)

prodEndpointId, prodBranchName, projectId, orgId, prodBranchId, defaultEnvFile, productionEnvFilesisModel.

## TESTS

(claude plugin test) covering at least: prod command held, confirmed twice, snapshot createroject-id/--org-id before the push, log and pane contents including the risk chip, the Markdownreport, the sql and rollback Code blocks, the copy button, and the 10-second auto-close; dev command passes untouched; unknown target treated as prod (DATABASE_URL=$X, bare psql -f, and the grep-into-variable recipe); wrong typed name cancels with no snapshot; dismissed dialog cancels; snapshot failure cancels with neonctl's error; no UI surface refuses; unconfigured endpoint holds everything; Neon MCP judged by branch_id with no branch_id as prod; /neon-guard resolve and check; analysis off skips the agent; the command analysis unit cases (false positives in echo/heredoc/comments, bash -c, eval, sudo env, npx, pnpm dlx, dotenv, psql heredoc); destructive SQL classification.

In tests: stub AskUserQuestion by the question's header; stub agent.spawn, agent.list (status completed, echoing the name), session.messages; mock.clock with a fixed time; process.run stubs for which/mkdir/cp/drizzle-kit/neonctl.

## ENGINE GOTCHAS LEARNED (so you don't hit them again)

* Every function that receives $ must be declared at the top level of the module, not nested inside register. Don't name a parameter or variable `on` or `h` (h is the JSX factory). atom() refs need literal plugin and key strings. Input is not in the mobile element table; narrow on e.surface. MultiEdit is not a tool name in this build. Uint8Array.fromBase64 is not available. tool.call matchers accept a RegExp. $.session.messages returns an array or a deny object. A stubbed agent.spawn in tests loses agentId. drizzle-kit generate --out must be a relative path and it turns interactive on suspected renames. neonctl prompts for the organization without --org-id. macOS zsh does not word-split unquoted variables in loops. Mods loaded from a session's dev-mods ettings.json pluginConfigs["neon-prod-guard@inline"].options.

## BUILD, VERIFY, TEST

* Run claude plugin validate (also --strict) and show me the hooks and calls it lists; type-checkd's declarations; all tests must pass.
* For a live test, configure a dev branch as "prod" (set prodEndpointId/prodBranchName/prodBranchues) and never test against real prod. Then explain how to install it permanently (CLAUDE_CODE_PLUGIN_DIRS or a marketplace)
