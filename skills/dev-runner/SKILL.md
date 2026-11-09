---
name: dev-runner
description: Analyze a code folder or repository and create simple root-level run.ps1 and run.sh scripts that start all components needed for local development, using live reload or hot reload where supported. Use whenever the user asks for a single command to run a project, a development launcher, startup scripts for a full stack or monorepo, or matching PowerShell and Bash runners. Prefer existing project commands and built-in tools; ask before installing any new tool.
---

# Development runner

Create two equivalent entry points, `run.ps1` and `run.sh`, in the target project root. Each starts the entire local development stack with one command. Keep the result understandable and easy to edit, not a new orchestration framework.

## Guardrails

- Use existing project scripts, framework development servers, runtime watch modes, shell features, and installed tools first.
- **Ask the user before installing any new tool.** This includes runtimes, package managers, CLIs, global packages, watch utilities, process managers, Docker, and automatic download mechanisms such as `npx`, `pnpm dlx`, or Corepack fetching a missing manager. Explain what is missing, why it helps, and any no-install alternative. Wait for approval before installing.
- Treat dependency installation as a separate setup step. Do not silently run restores or dependency installs while investigating or validating. Explain necessary setup and get approval before executing commands that download dependencies. Warn about development commands that implicitly restore packages or fetch toolchains.
- Generated launchers should check prerequisites and print actionable setup instructions, not automatically install tools or dependencies. If a native command can implicitly restore, document that behavior and use its no-restore/offline option when the project supports it and dependencies are already prepared.
- Do not add a process-manager dependency merely to run several commands. Reuse an existing workspace runner when it already handles the complete stack.
- Do not introduce Docker to a non-containerized project just for launching it. Reuse existing development Compose services if applicable and Docker is available.
- Do not change application behavior, rewrite manifests, expose development services publicly, commit secrets, reset databases, run destructive migrations, or kill unrelated processes.
- Read existing launchers before changing them. Preserve custom behavior and user edits. Ask before replacing nontrivial conflicting scripts.
- Produce both scripts unless the user explicitly requests only one. Match behavior across platforms, allowing honest, documented platform differences.

## 1. Discover the development stack

1. Resolve the requested folder. If none is specified, use the current project root. Read applicable repository instructions and development documentation.
2. Inspect relevant manifests, workspace definitions, lockfiles, example environment files, and existing launch scripts. Skip generated output, vendor directories, dependency trees, and secrets. Use targeted searches rather than reading the entire repository.
3. Identify independently runnable components: frontends, APIs, worker processes, and required local infrastructure. Libraries, test suites, production deployments, and one-off maintenance jobs are not additional servers.
4. Check whether the root already has a reliable command that starts the complete development stack. Wrapping that command in the two launchers is usually the simplest solution; do not also start its children separately.
5. Record each component's directory, exact development command, required executable, environment requirements, configured port, and startup dependencies. Distinguish long-running services from required one-shot preparation commands.
6. Follow the repository's selected package manager, lockfile, local virtual environment, wrappers, and documented runtime versions. Check availability without installing anything or contacting a package registry.
7. Resolve ambiguity only when it affects correctness: multiple unrelated applications, several mutually exclusive backends, missing infrastructure, or unclear setup. Otherwise proceed using repository evidence and state assumptions.

Give the user a short component/command summary. If a new tool is necessary, pause for approval and offer a simpler alternative where possible. If the code is only a library or has no runnable target, explain that rather than inventing a service.

## 2. Choose development commands

Prefer existing commands over inferred ones. These are examples, not commands to blindly impose:

| Stack | Preferred approach |
| --- | --- |
| Node/web frameworks | Existing `dev`/`start:dev` script using the repository's package manager; framework-provided HMR or reload |
| Plain Node | Existing watch script or installed runtime's native `node --watch` when supported |
| .NET | `dotnet watch run --project <actual-project>` with repository configuration |
| Python | Existing virtual environment and documented framework reload command, e.g. `python -m uvicorn ... --reload` or Flask debug mode |
| Django | Existing `manage.py runserver`, which already reloads by default |
| Ruby/Rails | Existing `bin/dev` or development server and its native reload behavior |
| Java/JVM | Repository's Maven/Gradle wrapper and documented development task; use existing framework reload support |
| Go/Rust/native code | Existing development/watch command if available; otherwise run normally and document that restart is manual |
| Containerized infrastructure | Existing development Compose configuration; do not assume production Compose is suitable |

Do not guess module names, executable targets, framework options, or ports. Verify them in project files or installed-command help. Do not assume Go, Rust, or JVM applications have built-in hot reload. If no native or already-installed watcher exists, use normal execution and explain the limitation; ask before adding a watcher.

Run every required development component, but only once. Include existing local infrastructure when needed; describe external services and credentials that cannot safely be launched locally. Never put actual secret values into scripts. Respect the project's environment-loading conventions; do not blindly source `.env` as shell code or assume Bash and PowerShell parse it identically.

## 3. Generate minimal, dependable scripts

Use explicit commands and a small amount of lifecycle handling. Avoid configuration generators, custom CLIs, extra helper files, elaborate argument parsing, or a generic orchestration abstraction. A short component table/helper is fine when it genuinely reduces repetition.

### Common behavior

- Resolve paths relative to the script location, not the caller's working directory. Support paths containing spaces.
- Start each component in its own required working directory. Preserve arguments correctly; avoid string concatenation plus `eval`/`Invoke-Expression`.
- Check required executables, component paths, and obvious setup prerequisites before launching anything. Report what is missing and how to prepare it, then exit nonzero.
- Show component names, commands, known local URLs, and how to stop the stack. Keep output visible; add log labels only when straightforward with existing tools. Do not detach into hidden terminals or background daemons.
- Start independent long-running components concurrently. Use existing readiness mechanisms or a small bounded readiness check only for actual dependencies. Prefer checking readiness to arbitrary sleeps; fail clearly on timeout.
- Do not run installs, destructive setup, tests, or production builds on every launch. Include a necessary safe one-shot build only when repository evidence requires it and report failures before starting dependent services.
- Keep the launcher alive while the stack runs. Detect unexpected service exits, including an early zero exit from a component intended to remain running; report the component, stop the others, and return nonzero. Propagate preparation failures.
- On Ctrl+C, termination, or startup failure, clean up processes started by this invocation, including child processes of package-manager/framework commands. Never use global name-based killing or terminate a preexisting service to free a port.
- For reused container infrastructure, do not stop services that were already running or delete volumes. Use the simplest existing lifecycle command; explicitly document any services intentionally left running.
- Do not automatically restart failed processes; native hot-reload servers already manage code-change restarts.

### `run.sh`

- Target Bash explicitly with `#!/usr/bin/env bash`, not an accidentally Bash-specific `/bin/sh` script.
- Derive the root from `BASH_SOURCE[0]`; quote paths and use arrays or direct commands for arguments.
- Use subshell working directories, tracked child PIDs, `wait`, and traps as appropriate. Remember that killing a wrapper PID alone may leave descendants alive; use a safe platform-supported ownership/process-group strategy and verify it.
- Avoid requiring a newer Bash feature such as `wait -n` if claiming support for macOS's default Bash. Avoid relying on `setsid`, GNU-only utilities, or other platform-specific features without checking availability and documenting the supported platforms.
- Make executable where supported. `bash ./run.sh` should also work.

### `run.ps1`

- Derive the root from `$PSScriptRoot` and use `Join-Path` or equivalent safe path handling.
- Prefer Windows PowerShell 5.1 compatibility unless the repository requires PowerShell 7. State any higher requirement rather than making the user install it silently.
- Use native command invocation or tracked process handles with visible output and explicit working directories. If using `Start-Process`, account for argument quoting and executable resolution, including `.cmd` package-manager shims on Windows.
- Use `try`/`finally` for owned-process cleanup and check native exit codes explicitly; `$ErrorActionPreference = 'Stop'` alone does not make native command failures throw in Windows PowerShell 5.1.
- Use an ownership-scoped process-tree cleanup mechanism supported on the target OS. Do not assume `Stop-Process` kills descendants or `taskkill` exists on non-Windows PowerShell.
- Do not change machine execution policy. If execution is blocked, explain a session-scoped invocation option rather than modifying persistent security settings.

If a platform cannot run a component natively, do not fake parity. Use an existing supported alternative such as WSL only if available and appropriate, or identify the limitation and ask how the user wants to proceed.

## 4. Verify without surprising the user

1. Review that both launchers run the same stack with the same development intent and no unnecessary dependencies.
2. Use installed tools for syntax checks: `bash -n run.sh` and PowerShell's parser API. Do not install a linter just to check the scripts.
3. When setup is ready and execution is safe, smoke-test startup, visible logs, and known endpoints. Test launching from a different working directory, missing-prerequisite failures, component failure, and Ctrl+C cleanup. Confirm no owned descendants remain.
4. Do not run commands that download tools/dependencies or cause unapproved persistent changes just to validate. If a runtime or environment is unavailable, say which checks could not be performed; never claim both platforms were tested when only one was.
5. Keep verification proportionate. Fix concrete issues rather than adding features or building a test framework for two scripts.

## Completion response

Briefly report:
- Created/updated `run.ps1` and `run.sh`, with their paths.
- Commands to use: `./run.ps1` and `bash ./run.sh` (or `./run.sh` when executable).
- Components started, known URLs, and which components have live/hot reload.
- Prerequisites or approved setup still needed, plus any platform or reload limitations.
- Checks actually performed and any unverified behavior.

The outcome is a boring, readable pair of development launchers—not a deployment system.
