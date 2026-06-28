# claude-codex-sync-aws

Back up raw Claude Code and Codex CLI chat history to AWS S3 without
client-side encryption.

The script uses `aws s3 sync` and `aws s3 cp` without `--delete`. That means:

- new files are uploaded;
- unchanged files are skipped by AWS CLI's normal sync logic;
- changed files are uploaded again;
- local deletes are not mirrored to S3;
- with S3 Versioning enabled, overwritten object versions remain recoverable.

This is a file-level incremental backup, not a tarball snapshot. It keeps the
remote objects readable as normal S3 objects.

## Tradeoffs

This tool intentionally does not use restic encryption or a restic password.
That makes restores simpler and keeps the backup layout transparent, but it
also means anyone with S3 read access to the bucket can read the raw logs.

File-level incremental backup is efficient for Claude/Codex JSONL session files,
which are mostly append-only or newly created. Large sqlite files are copied from
consistent Python sqlite snapshots and uploaded under stable S3 keys; if they
change, AWS CLI uploads the whole sqlite snapshot, and S3 Versioning keeps the
previous version.

If you need chunk-level deduplication for large changing files, use restic or
kopia instead; those tools require repository encryption passwords.

## What It Backs Up

Codex:

- `~/.codex/sessions/`
- `~/.codex/history.jsonl`
- `~/.codex/session_index.jsonl`
- `~/.codex/attachments/`
- `~/.codex/memories/`
- `~/.codex/shell_snapshots/`
- consistent sqlite snapshots of `state_5.sqlite`, `logs_2.sqlite`,
  `goals_1.sqlite`, and `memories_1.sqlite`

Claude:

- `~/.claude/projects/`
- `~/.claude/file-history/`
- `~/.claude/plans/`
- `~/.claude/commands/`
- `~/.claude/backups/`

It intentionally does not back up `~/.codex/auth.json`, caches, temp
directories, Codex worktrees, Claude paste caches, or `~/.claude.json`.

## Install

Install AWS CLI v2 first, then install the weekly user timer:

```bash
./bin/claude-codex-sync-aws install-user-service
```

The timer runs weekly on Sunday around 03:30, with up to two hours of randomized
delay and `Persistent=true` so a missed run is triggered after the machine comes
back.

## Configure AWS S3

Create or choose a private bucket and enable S3 Versioning. Object Lock is
optional.

Configure:

```bash
mkdir -p ~/.config/claude-codex-sync-aws
cp examples/env.example ~/.config/claude-codex-sync-aws/env
$EDITOR ~/.config/claude-codex-sync-aws/env
chmod 600 ~/.config/claude-codex-sync-aws/env
```

Minimum useful config:

```bash
AWS_S3_BUCKET=your-bucket-name
AWS_S3_PREFIX=claude-codex-sync-aws/raw
AWS_DEFAULT_REGION=us-east-1
AWS_PROFILE=your-profile
```

Set AWS credentials with one of:

- `aws configure`
- `aws sso login`
- an instance profile or role
- environment variables in the user service environment

## First Backup

```bash
claude-codex-sync-aws doctor
claude-codex-sync-aws check
claude-codex-sync-aws backup
claude-codex-sync-aws snapshots
```

## Restore

Restore the current remote prefix into a scratch directory:

```bash
claude-codex-sync-aws restore ~/restore-agent-history
```

For older overwritten versions, use the S3 console or `aws s3api
list-object-versions` / `get-object --version-id ...`.

## S3 Layout

Objects are written under:

```text
s3://$AWS_S3_BUCKET/$AWS_S3_PREFIX/$HOST/
```

Example:

```text
codex/sessions/...
codex/history.jsonl
codex/sqlite/logs_2.sqlite
claude/projects/...
manifests/latest.json
manifests/20260628T064500Z.json
```

## Operations

Service status:

```bash
systemctl --user status claude-codex-sync-aws.timer
systemctl --user status claude-codex-sync-aws.service
```

Logs:

```bash
journalctl --user -u claude-codex-sync-aws.service
```

Disable:

```bash
claude-codex-sync-aws uninstall-user-service
```

## Current Status

Small public utility. No license has been selected yet.
