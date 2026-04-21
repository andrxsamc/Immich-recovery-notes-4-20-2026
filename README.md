# Immich Recovery Notes

## What happened

Immich first started returning 404. After the stack came back, Immich loaded like a brand-new install and asked to create the first admin account again.

At first, it looked like one of these had failed:

- the VM itself
- Traefik
- Docker networking
- the 2TB media disk
- the Immich containers

After checking the stack, the actual media files were still present on disk, but the active Immich database metadata was no longer matching the library state.

The recovery worked by restoring Immich’s own SQL backup from the built-in backup folder.

---

## Important paths

- Immich media library:
  `/mnt/bulk/apps/immich/library`

- Immich Postgres data:
  `/mnt/bulk/apps/immich/db`

- Immich SQL backups:
  `/mnt/bulk/apps/immich/library/backups`

---

## What was confirmed before restore

### 1. The media files were still there

The library still existed on disk with a large number of files.

Commands used:

```bash
sudo find /mnt/bulk/apps/immich/library -type f | wc -l
sudo du -sh /mnt/bulk/apps/immich/library
sudo find /mnt/bulk/apps/immich/library -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort | tail -50
