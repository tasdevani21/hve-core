---
title: Test Scripts
description: Pester test runner, changed-file detection, and test directory organization
author: HVE Core Team
ms.date: 2026-08-03
ms.topic: reference
keywords:
  - powershell
  - pester
  - testing
  - automation
estimated_reading_time: 5
---

This directory contains the Pester test runner, changed-file detection
utilities, test configuration, and test suites organized to mirror the
production `scripts/` structure.

## Scripts

### `Invoke-PesterTests.ps1`

Pester test runner that writes structured output to `logs/`.

Purpose: Provide a consistent entry point for running Pester tests in both
local and CI environments.

#### Features

* Writes `logs/pester-summary.json` with overall pass/fail counts and duration.
  When code coverage is enabled, the summary also includes `CoveragePercent`
  (measured percentage, rounded to two decimal places) and `CoverageTarget`
  (configured threshold from `pester.config.ps1`)
* Writes `logs/pester-failures.json` with failure details including test name,
  file path, error message, and stack trace
* Supports code coverage reporting
* Integrates with CI for exit codes and NUnit output

#### Parameters

* `-TestPath` - Path to specific test file(s) or directory
* `-CI` (switch) - Enable CI mode with exit codes and NUnit output
* `-CodeCoverage` (switch) - Enable code coverage analysis
* `-Tag` / `-IncludeTag` - Run only tests whose Describe/Context/It blocks carry
  one of the supplied tags
* `-ExcludeTag` - Exclude tests whose blocks carry any of the supplied tags;
  defaults to `@('Integration','Slow')` when omitted, and passing this parameter
  (including `-ExcludeTag @()`) replaces the default rather than appending to it

#### Usage

```powershell
# Run all tests
./scripts/tests/Invoke-PesterTests.ps1

# Run a specific test directory
./scripts/tests/Invoke-PesterTests.ps1 -TestPath scripts/tests/linting/

# Run with code coverage in CI mode
./scripts/tests/Invoke-PesterTests.ps1 -CI -CodeCoverage
```

The corresponding npm script:

```bash
npm run test:ps
npm run test:ps -- -TestPath "scripts/tests/security/"
```

### `Get-ChangedTestFiles.ps1`

Detects changed PowerShell files and resolves corresponding Pester test paths.

Purpose: Enable targeted test runs by identifying which tests correspond to
changed production scripts.

#### Features

* Compares the current branch against a base branch using `git diff`
* Maps changed production files to their mirror test files
* Supports custom file filters and alternate root paths

#### Parameters

* `-BaseBranch` - Git branch to compare against (defaults to `main`)
* `-FileFilter` - Glob pattern for filtering changed files
* `-SkillsRoot` - Root path for skill scripts
* `-TestRoot` - Root path for test files

#### Usage

```powershell
# Get test files for all changed scripts
./scripts/tests/Get-ChangedTestFiles.ps1

# Compare against a specific branch
./scripts/tests/Get-ChangedTestFiles.ps1 -BaseBranch develop
```

### `pester.config.ps1`

Pester 5.x configuration script that defines test execution behavior, coverage
targets, and output paths. See
[Testing Architecture](../../docs/architecture/testing.md) for configuration
details.

## Directory Structure

Test suites mirror the production `scripts/` layout:

```text
tests/
├── collections/     Collection validation tests
├── extension/       Extension packaging tests
├── lib/             Library utility tests
├── linting/         Linting script tests
├── plugins/         Plugin generation tests
├── security/        Security validation tests
├── Fixtures/        Shared test fixtures
└── Mocks/           Shared mock data (GitMocks.psm1)
```

Test files use the `.Tests.ps1` suffix convention for automatic discovery by
Pester.

Planner-related tests are split intentionally: rule-validator suites (state
schema, cadence ordering, startup blocks, risk-grid grammar) live under
`linting/` alongside other linter tests, while artifact-signing and runtime
concerns (e.g. `Sign-PlannerArtifacts.Tests.ps1`) live under `security/`.

## PowerShell 7 and Pester Gotchas

These non-obvious traps have burned contributors more than once. Each has a
recommended pattern already in use in
`scripts/tests/linting/Test-Format-MarkdownTables.Tests.ps1`.

### `Get-Content -Raw` on a 0-byte file returns `$null`

In PowerShell 7, `Get-Content -Raw` on an empty file returns `$null`, not an
empty string. `Should -Match` against `$null` fails intermittently when the
subject under test happens to emit no output.

Read the file with `[System.IO.File]::ReadAllText` behind a size guard so an
empty file becomes `''`:

```powershell
$bytes = if (Test-Path $stdoutPath) { (Get-Item $stdoutPath).Length } else { -1 }
$stdout = if ($bytes -gt 0) { [System.IO.File]::ReadAllText($stdoutPath) } else { '' }
```

### `Start-Process -Wait` can return before the stdout file handle flushes

`Start-Process -Wait -RedirectStandardOutput` sometimes returns before the OS
finishes flushing the redirected file handle, especially for tiny payloads
under the Pester runspace. A subsequent read observes 0 bytes even though the
process exited cleanly.

Belt-and-suspenders: call `WaitForExit` on the returned process, check the file
size, and re-check after a short sleep before reading:

```powershell
$proc.WaitForExit()
$bytes = (Get-Item $stdoutPath).Length
if ($bytes -eq 0 -and $proc.ExitCode -eq 0) {
    Start-Sleep -Milliseconds 100
    $bytes = (Get-Item $stdoutPath).Length
}
```

### `[System.IO.File]::AppendAllText` inside an `It` block is unreliable

Diagnostic writes to a log file with `[System.IO.File]::AppendAllText` from
inside a Pester `It` block are sometimes silently swallowed, so in-test log
sinks are not a reliable diagnostic channel.

Use `Should -Because '<message>'` for diagnostic context. The `-Because` text
reliably surfaces in `logs/pester-failures.json` produced by the repository's
Pester test runner:

```powershell
$stdout | Should -Match 'formatted:' -Because "process exited $($proc.ExitCode) with $bytes bytes on stdout"
```

## Related Documentation

* [Testing Architecture](../../docs/architecture/testing.md) for Pester
  configuration and conventions
* [Scripts README](../README.md) for overall script organization

<!-- markdownlint-disable MD036 -->
*🤖 Crafted with precision by ✨Copilot following brilliant human instruction,
then carefully refined by our team of discerning human reviewers.*
<!-- markdownlint-enable MD036 -->
