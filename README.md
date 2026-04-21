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
```

This showed:

- about 81,916 files
- about 207G of media still present
- last confirmed original media files on disk from around 2026-04-16

### 2. The active Immich database looked wrong

The schema existed, but the main tables had no real data.

Commands used:

```bash
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'
```

At that time the results showed:

- `user` = 0
- `asset` = 0
- `library` = 0

That explained why Immich was asking for first-time setup again.

### 3. Built-in Immich SQL backups existed

Commands used:

```bash
sudo find /mnt/bulk/apps/immich/library/backups -type f
```

Backups found included:

- `immich-db-backup-20260416T020000-v2.7.5-pg14.19.sql.gz`
- `immich-db-backup-20260417T020000-v2.7.5-pg14.19.sql.gz`
- `immich-db-backup-20260418T020000-v2.7.5-pg14.19.sql.gz`

---

## Recovery steps

### Step 1. Stop Immich-related containers

```bash
cd /opt/docker
docker compose stop immich immich-machine-learning immich_postgres immich_redis
```

### Step 2. Make a backup copy of the current DB folder before touching anything

```bash
sudo cp -a /mnt/bulk/apps/immich/db /mnt/bulk/apps/immich/db-before-restore-$(date +%F-%H%M)
```

### Step 3. Start only Postgres

```bash
cd /opt/docker
docker compose up -d immich_postgres
docker compose ps
```

### Step 4. Drop and recreate the Immich database

```bash
docker exec -it immich_postgres psql -U postgres -d postgres -c 'DROP DATABASE IF EXISTS immich;'
docker exec -it immich_postgres psql -U postgres -d postgres -c 'CREATE DATABASE immich;'
```

### Step 5. Restore the latest known-good Immich SQL backup

```bash
gunzip -c /mnt/bulk/apps/immich/library/backups/immich-db-backup-20260418T020000-v2.7.5-pg14.19.sql.gz | docker exec -i immich_postgres psql -U postgres -d immich
```

### Step 6. Verify the restore actually brought back the account and assets

```bash
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select email from "user";'
```

After restore, the DB showed:

- `user` = 1
- `asset` = 21867
- restored email:
  `andresarturomarquez@gmail.com`

### Step 7. Start Immich again

```bash
cd /opt/docker
docker compose up -d immich immich-machine-learning immich_redis
```

### Step 8. Verify the app and proxy were healthy again

```bash
cd /opt/docker
docker compose ps
docker logs immich --tail 100
docker logs traefik --tail 100 | grep -i immich
curl -k -I https://127.0.0.1 -H 'Host: immich.stamperstack.win'
```

That final curl returned HTTP 200, confirming Traefik and Immich were working again.

---

## Final result

The restore recovered:

- the original Immich account
- about 21,867 assets
- thumbnails and normal UI state

The media files on disk had survived the whole time.

Anything after the restored backup point was not recovered in metadata.

---

## Commands used in one block

```bash
cd /opt/docker
docker compose stop immich immich-machine-learning immich_postgres immich_redis

sudo find /mnt/bulk/apps/immich/library -type f | wc -l
sudo du -sh /mnt/bulk/apps/immich/library
sudo find /mnt/bulk/apps/immich/library -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort | tail -50

sudo find /mnt/bulk/apps/immich/library/backups -type f

docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'

sudo cp -a /mnt/bulk/apps/immich/db /mnt/bulk/apps/immich/db-before-restore-$(date +%F-%H%M)

cd /opt/docker
docker compose up -d immich_postgres
docker compose ps

docker exec -it immich_postgres psql -U postgres -d postgres -c 'DROP DATABASE IF EXISTS immich;'
docker exec -it immich_postgres psql -U postgres -d postgres -c 'CREATE DATABASE immich;'

gunzip -c /mnt/bulk/apps/immich/library/backups/immich-db-backup-20260418T020000-v2.7.5-pg14.19.sql.gz | docker exec -i immich_postgres psql -U postgres -d immich

docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select email from "user";'

cd /opt/docker
docker compose up -d immich immich-machine-learning immich_redis

cd /opt/docker
docker compose ps
docker logs immich --tail 100
docker logs traefik --tail 100 | grep -i immich
curl -k -I https://127.0.0.1 -H 'Host: immich.stamperstack.win'
```
