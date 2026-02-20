# [PACKAGE NAME] - CLI Tool

[One-sentence description of what this CLI does and the problem it solves.]

## Language & Runtime

- Language: [TypeScript / Python 3.12+]
- Runtime: [Node.js 20+ / Python 3.12+]
- CLI Framework: [Commander.js / yargs / Click / Typer]
- Package Manager: [npm / pnpm / pip / uv]

## Project Structure

```
[package-name]/
  src/
    cli.ts              # Entry point, argument parsing, command registration
    commands/           # One file per command (e.g., init.ts, build.ts, deploy.ts)
    lib/                # Core logic (framework-agnostic, importable as library)
    utils/              # Shared utilities (logging, config, formatting)
    types.ts            # Shared TypeScript types / Python dataclasses
  bin/
    [package-name]      # Executable entry (shebang + import src/cli)
  tests/
    unit/               # Pure logic tests (no filesystem, no network)
    integration/        # Tests that touch filesystem or spawn processes
    fixtures/           # Test input files, expected outputs
    snapshots/          # CLI output snapshots
  dist/                 # Build output (gitignored)
  [package-name].config.ts  # Default config schema example
```

## CLI Commands and Arguments

```bash
[package-name] [command] [arguments] [options]

# Core commands
[package-name] init [project-name]        # Scaffold a new project
[package-name] [verb] [target]            # Primary action
[package-name] [verb] [target] --flag     # With options

# Global options (available on every command)
--config <path>       # Custom config file path
--verbose, -v         # Increase log verbosity (stackable: -vvv)
--quiet, -q           # Suppress non-error output
--no-color            # Disable colored output (also honors NO_COLOR env var)
--json                # Output machine-readable JSON instead of human text
--version             # Print version and exit
--help, -h            # Print help and exit
```

## Config File Resolution

Config is loaded in this precedence order (first found wins):
1. `--config <path>` flag (explicit)
2. `[package-name].config.ts` in current directory
3. `[package-name].config.json` in current directory
4. `.[package-name]rc` in current directory
5. `[package-name]` key in `package.json` (Node.js) / `pyproject.toml` (Python)
6. `~/.config/[package-name]/config.json` (global user config)
7. Built-in defaults

## Exit Codes

| Code | Meaning | When |
|------|---------|------|
| 0 | Success | Command completed normally |
| 1 | General error | Unhandled exception, unknown failure |
| 2 | Usage error | Invalid arguments, missing required options |
| 3 | Config error | Config file not found, invalid schema, parse failure |
| 4 | Runtime error | Network failure, permission denied, resource unavailable |
| [5+] | [Domain-specific] | [e.g., lint violations found, tests failed] |

Always exit with a code. Never `process.exit()` / `sys.exit()` from deep in the call stack --
propagate errors up and exit from the CLI entry point.

## Output Conventions

```
# Human output (default)
- Progress: use spinners/progress bars for operations > 1s ([ora / cli-spinners / rich])
- Success: green checkmark + summary message
- Warnings: yellow prefix, written to stderr
- Errors: red prefix + actionable message + suggested fix, written to stderr
- Tables: use [cli-table3 / rich.table] for structured data

# Machine output (--json flag)
- All output is valid JSON to stdout
- Errors are JSON objects to stderr: {"error": "message", "code": "ERROR_CODE"}
- No spinners, colors, or interactive elements
```

## Error Handling

```
Every user-facing error MUST include:
1. What went wrong (specific, not "An error occurred")
2. Why it likely happened (most common cause)
3. How to fix it (concrete command or action)

Example:
  Error: Config file not found at ./[package-name].config.ts
  Run `[package-name] init` to create one, or use --config to specify a path.
```

## Build and Bundle

```bash
# TypeScript
npm run build                   # tsc or tsup -> dist/
npm run dev                     # Watch mode (tsx or tsup --watch)

# Python
uv build                        # Build wheel + sdist
pip install -e ".[dev]"         # Editable install with dev deps

# Verify the CLI works after build
node dist/cli.js --version      # TypeScript
python -m [package_name] --version  # Python
```

**Shebang line** (bin/[package-name]):
```
#!/usr/bin/env node
```
This file must have execute permissions (`chmod +x`). Git tracks this via `git update-index --chmod=+x`.

## Testing

```bash
# Run tests
npm test                        # or: pytest
npm run test:unit               # Unit tests only
npm run test:integration        # Integration tests (may need fixtures)
npm run test:snapshots          # CLI output snapshot tests

# Update snapshots after intentional output changes
npm run test:snapshots -- --update  # or: pytest --snapshot-update
```

**Testing strategy:**
- **Unit tests**: All `lib/` functions. Pure input->output, no mocking needed if lib is well-separated.
- **Integration tests**: Spawn the actual CLI as a child process, capture stdout/stderr/exit code.
  Use a temp directory for filesystem operations. Clean up after every test.
- **Snapshot tests**: Capture `--help` output and key command outputs. Catches unintentional
  formatting regressions. Keep snapshots small and focused.
- **No mocking the CLI framework**: Test commands through the real entry point, not by importing
  and calling handler functions directly (that misses argument parsing bugs).

## Publishing

```bash
# Pre-publish checklist
# 1. All tests pass
# 2. Version bumped in package.json / pyproject.toml (semver: MAJOR.MINOR.PATCH)
# 3. CHANGELOG.md updated
# 4. Build output is fresh (npm run build / uv build)

# npm
npm publish                     # Publish to npm registry
npm publish --dry-run           # Preview what would be published
npm pack                        # Create tarball for inspection

# PyPI
uv publish                      # Publish to PyPI
twine upload dist/*             # Alternative: twine
twine upload --repository testpypi dist/*  # Test on TestPyPI first
```

**Versioning**: Follow semver strictly. CLI flag changes (rename, remove) are MAJOR. New commands
or flags are MINOR. Bug fixes and output formatting changes are PATCH.

## Common Gotchas

1. **ESM vs CJS (Node.js)**: If `package.json` has `"type": "module"`, all `.js` files are ESM.
   Use `.cjs` extension for any CommonJS files. `__dirname` and `require()` are NOT available in
   ESM -- use `import.meta.url` and `import.meta.dirname` (Node 21+) or
   `fileURLToPath(import.meta.url)`. tsup/esbuild can bundle to CJS if needed for compatibility.

2. **Shebang + Windows**: `#!/usr/bin/env node` works on Windows via npm/npx, but NOT if the user
   runs the file directly. The `bin` field in `package.json` handles cross-platform dispatch.
   For Python: use `entry_points` / `[project.scripts]` in pyproject.toml, never raw shebangs.

3. **Global install paths**: `npm install -g` puts binaries in a user-specific path that may not
   be in `$PATH`. Your README should mention `npx [package-name]` as the zero-install alternative.
   For Python: recommend `pipx install [package-name]` for isolated global installs.

4. **stdin/stdout handling**: Detect if stdin is a TTY (`process.stdin.isTTY` / `sys.stdin.isatty()`)
   before showing interactive prompts. When piped, read stdin as input data instead. Always write
   human-readable output to stdout and errors/warnings to stderr -- this allows `cmd | other-cmd`
   piping to work correctly.

5. **NO_COLOR and FORCE_COLOR**: Respect the `NO_COLOR` env var (https://no-color.org/).
   Most color libraries (chalk, kleur, rich) handle this automatically, but verify. CI environments
   often set `CI=true` -- disable interactive features (spinners, prompts) when detected.

6. **Config file TypeScript imports**: If your config supports `.ts` files, you need a loader
   (tsx, jiti, or bundle the config schema as JSON). Users without TypeScript installed will get
   cryptic errors if you try to import `.ts` directly.

7. **Large output and paging**: For output longer than ~50 lines, consider piping through a pager
   (`less`) automatically when stdout is a TTY. Or provide a `--no-pager` flag.

8. **Symlink issues**: `npm link` during development creates symlinks that behave differently
   from a real install (especially for peer dependencies and native modules). Always test with
   `npm pack && npm install -g ./[package]-[version].tgz` before publishing.
