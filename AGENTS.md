# AGENTS.md — SQLite Backup Tool

## Project Shape

Single-file Python utility (`backup.py`, ~580 lines). No package structure, no test suite, no CI, no linting/formatting config. One external dependency: `pyyaml>=6.0.1`.

## Quick Commands

```bash
# Install
pip install -r requirements.txt

# Run with default config (config.yaml in cwd)
python backup.py

# Dry-run to validate config without side effects
python backup.py --config my-config.yaml --dry-run --verbose

# Full CLI mode (no config file)
python backup.py --source ./app.db --backup-dir ./backups --keep 7

# Check version
python backup.py --version
```

## Architecture Notes

- **Entrypoint**: `backup.py` only. Everything is self-contained.
- **Config**: YAML, default path `config.yaml`. Can be overridden with `-c`.
- **Python**: Requires 3.7+ (uses `sqlite3.Connection.backup()` online backup API).
- **Atomic writes**: The tool always backs up to a temp file in `backup_dir`, then `os.rename()` to the final name. Do not change this — it’s the corruption-safety guarantee.
- **Online backup API**: Uses `source_conn.backup(target_conn, pages=5)`, not `shutil.copy()`. This allows concurrent reads/writes on the source DB during backup.
- **Verification**: Opens backup with `file:…?mode=ro` URI and runs `PRAGMA integrity_check`. Failed backups are automatically deleted.
- **Retention cleanup**: Matches filenames with a glob derived from the strftime format; sorts by mtime; deletes oldest when exceeding `retention_count`.

## Development / Testing Reality

- **No tests exist** in the repo. The README mentions `python -m pytest`, but there is no `pytest` dependency and no test files.
- **No lint/typecheck/format config** (no `pyproject.toml`, `setup.cfg`, `.flake8`, etc.).
- If adding tests, create them from scratch. A minimal smoke test would:
  1. Create a temp SQLite DB.
  2. Run `backup.py --source … --backup-dir … --keep 1`.
  3. Assert the backup file exists and `PRAGMA integrity_check` passes.

## Exit Codes (Important for Cron)

The script returns specific exit codes. Any non-zero should trigger cron alerts:

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Config error |
| 2 | Source DB error |
| 3 | Backup directory error |
| 4 | Backup operation failed |
| 5 | Verification failed |
| 6 | Cleanup warning (non-fatal) |

## What Not to Break

- **Temp-then-rename atomicity** in `main()`.
- **Source DB lock behavior**: The backup API needs a normal (not read-only URI) connection to the source. Do not switch to `mode=ro` for the source or the online backup will fail.
- **Cleanup on failure**: Partial backups and failed verifications are deleted before exit.

## Existing Instruction Files

- `README.md` — Full user-facing docs, config reference, cron examples.
- `examples/config.example.yaml` — Production-style config template.
- `examples/crontab.example` — Ready-to-use cron schedule examples.
