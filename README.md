[![Molecule](https://github.com/iamenr0s/ansible-role-logging-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-logging-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/logging_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-logging-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-logging-hardening)

# Ansible Role: Logging Hardening

> Standalone copy of `logging_hardening` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables keep the collection-wide `linux_hardening_*` prefix.

Hardens log storage through drop-ins; no existing logging configuration is edited.

## What it enforces
| File | Setting | Default |
|---|---|---|
| `/etc/systemd/journald.conf.d/99-linux-hardening.conf` | `Storage=persistent`, `Compress=yes` | on |
| `/etc/rsyslog.d/99-linux-hardening.conf` | `$FileCreateMode 0640` | only when rsyslog is installed (`/etc/rsyslog.conf` exists) |

Add or change journald settings with `linux_hardening_logging_journald` (e.g. `{SystemMaxUse: 2G}`; a `null` value drops a built-in one). Change the rsyslog mode with `linux_hardening_logging_rsyslog_file_mode`.

## Modes
`audit` (default) is read-only and sets `linux_hardening_logging_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`.

## Safety
- Remediation restarts `systemd-journald`, and restarts rsyslog only if it is running and its drop-in changed. A journald restart does not lose already-stored logs.
- `Storage=persistent` uses disk under `/var/log/journal`; cap it with `SystemMaxUse` on small disks.
- `$FileCreateMode` affects only files rsyslog creates later; existing log files keep their mode (see `filesystem_hardening` for system files).

## Role Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_hardening_mode` | str | `audit` | audit is read-only. Choices: `audit`, `remediate`. |
| `linux_hardening_confirm_remediation` | bool | `false` | Must be true before remediate changes anything. |
| `linux_hardening_backup` | bool | `true` | Back up a drop-in when it changes. |
| `linux_hardening_verify` | bool | `true` | Re-audit after remediation. |
| `linux_hardening_fail_on_error` | bool | `true` | Fail the play when audit finds deviations. |
| `linux_hardening_logging_enabled` | bool | `true` | Run the role. |
| `linux_hardening_logging_journald_dropin` | path | `/etc/systemd/journald.conf.d/99-linux-hardening.conf` | journald drop-in. |
| `linux_hardening_logging_rsyslog_dropin` | path | `/etc/rsyslog.d/99-linux-hardening.conf` | rsyslog drop-in. |
| `linux_hardening_logging_journald` | dict | `{}` | [Journal] settings to add or change; null drops a built-in one. |
| `linux_hardening_logging_rsyslog_file_mode` | str | `0640` | rsyslog $FileCreateMode. |

**Check mode:** `remediate` with `--check --diff` shows what would change without writing anything; the post-remediation re-audit is skipped in check mode.

**Idempotent:** yes. A second `remediate` run reports no changes (the Molecule `idempotence` step checks this).

## Example
```yaml
- hosts: all
  become: true
  roles:
    - role: iamenr0s.logging_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_logging_journald: {SystemMaxUse: 2G}
```

## Verification
Molecule `default` (remediate, idempotence, re-audit, journald still active, a stubbed `/etc/rsyslog.conf` exercises the rsyslog drop-in), `audit` (writes nothing), `refuse`. Tested: AlmaLinux 9 and Debian 13 (default); AlmaLinux 9 (audit, refuse).

## Rollback
Delete the two drop-ins and restart journald (backups of earlier versions are kept when `linux_hardening_backup` is true).

## Known gaps
- rsyslog was not installed in the test images, so its restart path was not exercised against a real daemon.
- No remote log forwarding, log rotation, or `/var/log` permission changes. CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-logging-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-logging-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_logging_hardening`
