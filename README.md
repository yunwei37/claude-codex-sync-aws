# claude-codex-sync-aws

Back up raw Claude Code and Codex CLI chat history to AWS S3 with
[restic](https://restic.net/).

The goal is durable backup, not directory mirroring. If a local chat record is
deleted later, older restic snapshots can still restore it. The script never
runs `restic forget`, `restic prune`, or `aws s3 sync --delete`.

## Why restic, not borg?

Use `restic` for AWS S3. Restic supports S3 directly, encrypts data locally,
deduplicates repeated content, and stores point-in-time snapshots. `borg` is
excellent for SSH targets such as BorgBase, rsync.net, or a VPS, but it does not
write directly to S3.

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

Install restic first:

```bash
sudo apt-get update
sudo apt-get install -y restic
```

Install the script and weekly user timer:

```bash
./bin/claude-codex-sync-aws install-user-service
```

The timer runs weekly on Sunday around 03:30, with up to two hours of randomized
delay and `Persistent=true` so a missed run is triggered after the machine comes
back.

## Configure AWS S3

Create or choose a bucket, then configure:

```bash
mkdir -p ~/.config/claude-codex-sync-aws
cp examples/env.example ~/.config/claude-codex-sync-aws/env
$EDITOR ~/.config/claude-codex-sync-aws/env
chmod 600 ~/.config/claude-codex-sync-aws/env
```

Minimum useful config:

```bash
AWS_S3_BUCKET=your-bucket-name
RESTIC_REPOSITORY_PREFIX=claude-codex-sync-aws
AWS_DEFAULT_REGION=us-east-1
RESTIC_PASSWORD_FILE=${HOME}/.config/claude-codex-sync-aws/restic-password
```

Set AWS credentials with one of:

- `aws configure`
- `aws sso login`
- an instance profile or role
- environment variables in the user service environment

Avoid committing credentials or the restic password. If the password is lost,
the backup cannot be restored.

## First Backup

```bash
claude-codex-sync-aws doctor
claude-codex-sync-aws backup
claude-codex-sync-aws snapshots
```

The first run initializes the restic repository if needed. Later runs are
incremental.

## Restore

Restore the latest snapshot into a scratch directory:

```bash
claude-codex-sync-aws restore ~/restore-agent-history latest
```

List snapshots:

```bash
claude-codex-sync-aws snapshots
```

## S3 Object Lock

For stronger protection against accidental or malicious deletion, create the S3
bucket with versioning and Object Lock before the first backup. Object Lock
configuration is intentionally left outside this script because compliance-mode
retention is hard to undo during the retention window.

The script itself only creates new restic snapshots. It does not delete older
snapshots.

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
