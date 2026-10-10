---
title: "Tanzu Platform お引越しメモ (Bosh Directorまで）"
date: "2026-09-10T19:00:00+09:00"
tags: ["Tanzu Platform", "Backup", "Restore", ""]
thumbnail: img.png
draft: true
---


# BOSH Director Backup Procedure (CLI)

Reference notes for backing up the BOSH Director via `bbr` (BOSH Backup and Restore),
including how to free up blobstore space beforehand if the director's persistent disk
is too full to complete a backup.

## Environment facts

- Ops Manager `om` target: `https://pivotal-ops-manager.vpsmart.aws.lespaulstudioplus.info`
- BOSH Director `om` installation_name: `p-bosh`
- BOSH Director IP: `192.168.200.2`
- `bbr` SSH username: `bbr`
- Credential reference (for the bbr SSH key): `.director.bbr_ssh_credentials`
  (fields: `private_key_pem`, `public_key_pem`)

### CLI aliases (from project CLAUDE.md)

```bash
alias bosh="BOSH_CLIENT=ops_manager BOSH_CLIENT_SECRET=<client-secret> BOSH_CA_CERT=/var/tempest/workspaces/default/root_ca_certificate BOSH_ENVIRONMENT=192.168.200.2 bosh"
```

`om` config (`env.yaml`):

```yaml
target: https://pivotal-ops-manager.vpsmart.aws.lespaulstudioplus.info
connect-timeout: 30
request-timeout: 1800
skip-ssl-validation: true
username: admin
password: "<password>"
decryption-passphrase: "<password>"
```

Actual secret values live in the project's `CLAUDE.md` — don't duplicate them elsewhere.

## 1. Check disk space before attempting a backup

`bbr` streams the director's blobstore (`/var/vcap/store/blobstore`) plus small DB
dumps (director db, credhub db, uaa db, postgres) down to wherever you run `bbr` from.
**The destination needs free space roughly equal to the current blobstore size** —
uncompressed, and bbr does not stream-compress to disk as it goes.

```bash
# Local destination disk
df -h .

# Remote director disk usage
ssh -i bbr_ssh_key.pem bbr@192.168.200.2 \
  "sudo df -h /var/vcap/store; sudo du -sh /var/vcap/store/*"
```

If free space is tight relative to blobstore size, do the cleanup in step 2 first.

## 2. (Optional) Shrink the blobstore with `bosh clean-up`

Unused releases, stemcells, and compiled packages accumulate on the director over
time (frequent upgrades, stemcell churn, tile redeploys) and are not removed
automatically. See: https://knowledge.broadcom.com/external/article/428186

```bash
# See what would be removed, without deleting anything
bosh clean-up --dry-run -n --tty --no-color

# Conservative cleanup: removes unused resources, keeps orphaned disks and the
# 2 most recent stemcell/release versions
bosh clean-up -n

# Aggressive cleanup: removes ALL unreferenced releases/stemcells/compiled
# packages, but still keeps orphaned disks
bosh clean-up --all --keep-orphaned-disks -n

# Most aggressive: also removes orphaned disks
bosh clean-up --all -n
```

Notes:
- `clean-up` only ever removes resources that are *unreferenced* by any current
  deployment — it won't touch anything actually in use, so it may report nothing
  to clean up if the blobstore is legitimately full of in-use artifacts.
- After `--all`, disk usage may take a short while to settle (async reclaim) —
  re-run `df -h` / `du -sh` a minute later if numbers look inconsistent.
- This is disruptive in the sense that removed release/stemcell versions can no
  longer be used to redeploy without re-uploading them — fine for versions no
  longer referenced by any deployment, but confirm that's the case for your
  environment before running `--all`.

## 3. Extract the bbr SSH private key from Ops Manager

```bash
om -e env.yaml credentials -p p-bosh -c .director.bbr_ssh_credentials -t json \
  > bbr_ssh_creds.json

# Extract the private_key_pem field and write it out as a real PEM file
python3 -c "
import json
d = json.load(open('bbr_ssh_creds.json'))
open('bbr_ssh_key.pem', 'w').write(d['credential']['private_key_pem'])
"
chmod 600 bbr_ssh_key.pem
```

## 4. Pre-backup check

```bash
bbr director \
  --private-key-path bbr_ssh_key.pem \
  --username bbr \
  --host 192.168.200.2 \
  pre-backup-check
```

## 5. Run the backup

Run this in `tmux`/`nohup` or background it — it can take a long time depending on
blobstore size.

```bash
bbr director \
  --private-key-path bbr_ssh_key.pem \
  --username bbr \
  --host 192.168.200.2 \
  backup
```

This produces a timestamped directory (e.g. `192.168.200.2_YYYYMMDDThhmmssZ/`)
containing per-job backup archives (director, blobstore, credhub db, uaa db,
count-cores) plus a metadata file.

## 6. If the backup fails or is interrupted: clean up before retrying

`bbr` leaves locks and remote staging data (under `/var/vcap/store/bbr-backup` on
the director) if a backup run dies mid-way. Always run this before attempting
another backup:

```bash
bbr director \
  --private-key-path bbr_ssh_key.pem \
  --username bbr \
  --host 192.168.200.2 \
  backup-cleanup
```

This unlocks any jobs left locked (e.g. uaa, credhub) and removes remote temp
files. Safe to run any time; it's a no-op if the director is already clean.

## Troubleshooting reference

| Symptom | Cause | Fix |
|---|---|---|
| Backup fails partway with "no space left on device" | Destination disk smaller than blobstore | Free space (step 2), or run `bbr` from/to a host with more disk (blobstore-size + overhead) |
| Backup left director locked / next attempt errors about existing backup | Previous run died without cleanup | `bbr director ... backup-cleanup` before retrying |
| `bosh clean-up --dry-run` shows nothing removable | Blobstore is genuinely all in-use artifacts | Nothing to reclaim via clean-up; reduce deployed product/stemcell versions instead, or size the backup destination for the real blobstore size |
| `du -sh` numbers look inconsistent right after `clean-up --all` | Async disk reclaim still settling | Wait ~1 min and re-check |