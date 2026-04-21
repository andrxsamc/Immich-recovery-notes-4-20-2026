# Immich-recovery-notes-4-20-2026
# immich-recovery-notes

## Incident summary

Immich started returning 404 and later behaved like a fresh install, asking to create a new admin account again.

At first, it looked like the VM, Traefik, Docker, or the 2TB media disk had failed completely. After checking the stack, the media files were still present on disk, but the active Immich database metadata was not matching the real library state.

The final recovery worked by restoring Immich's own SQL backup from the built-in backup folder.

## Environment

- Proxmox host
- Debian Docker VM
- Traefik reverse proxy
- Immich
- Media library on:
  `/mnt/bulk/apps/immich/library`
- Immich Postgres data on:
  `/mnt/bulk/apps/immich/db`

## What was found

### Media files still existed
The library still had files on disk:

- around 81,916 files
- around 207G

### Active DB looked wrong
The active Immich DB had schema/tables, but at one point:

- `user` = 0
- `asset` = 0
- `library` = 0

That is why Immich asked for first-time setup again.

### Built-in Immich SQL backups existed
These were found here:

`/mnt/bulk/apps/immich/library/backups`

Example backups found:

- `immich-db-backup-20260416T020000-v2.7.5-pg14.19.sql.gz`
- `immich-db-backup-20260417T020000-v2.7.5-pg14.19.sql.gz`
- `immich-db-backup-20260418T020000-v2.7.5-pg14.19.sql.gz`

## Recovery result

Restoring the SQL backup from `2026-04-18` brought back:

- the original user account
- about 21,867 assets
- the normal Immich account state

Recovered email:

- `andresarturomarquez@gmail.com`

## Important lesson

The media files survived, but the active metadata DB did not reflect the real library anymore.

The built-in Immich SQL backup is what saved the recovery.

## Last confirmed media date on disk

Last confirmed original media files present on disk were from:

- `2026-04-16`

So anything after that date was probably not backed up into the recovered metadata state.

## Commands used

See `commands-used.md`.
