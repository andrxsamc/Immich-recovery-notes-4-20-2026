Use this as your repo note.

`README.md`

```md
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
```

`commands-used.md`

````md
# Commands used to recover Immich

## Stop Immich-related containers

```bash
cd /opt/docker
docker compose stop immich immich-machine-learning immich_postgres immich_redis
````

## Check current DB state

```bash
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'
```

## Verify library files still existed

```bash
sudo find /mnt/bulk/apps/immich/library -type f | wc -l
sudo du -sh /mnt/bulk/apps/immich/library
sudo find /mnt/bulk/apps/immich/library -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort | tail -50
```

## Check built-in Immich SQL backups

```bash
sudo find /mnt/bulk/apps/immich/library/backups -type f
```

## Make a backup copy of the current DB folder before restore

```bash
sudo cp -a /mnt/bulk/apps/immich/db /mnt/bulk/apps/immich/db-before-restore-$(date +%F-%H%M)
```

## Start only Postgres

```bash
cd /opt/docker
docker compose up -d immich_postgres
docker compose ps
```

## Drop and recreate the Immich DB

```bash
docker exec -it immich_postgres psql -U postgres -d postgres -c 'DROP DATABASE IF EXISTS immich;'
docker exec -it immich_postgres psql -U postgres -d postgres -c 'CREATE DATABASE immich;'
```

## Restore the SQL backup

```bash
gunzip -c /mnt/bulk/apps/immich/library/backups/immich-db-backup-20260418T020000-v2.7.5-pg14.19.sql.gz | docker exec -i immich_postgres psql -U postgres -d immich
```

## Verify restore worked

```bash
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from "user";'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from asset;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select count(*) from library;'
docker exec -it immich_postgres psql -P pager=off -U postgres -d immich -c 'select email from "user";'
```

## Start Immich again

```bash
cd /opt/docker
docker compose up -d immich immich-machine-learning immich_redis
```

## Check stack state

```bash
cd /opt/docker
docker compose ps
docker logs immich --tail 100
docker logs traefik --tail 100 | grep -i immich
curl -k -I https://127.0.0.1 -H 'Host: immich.stamperstack.win'
```

````

Here’s the command list in plain bash format too:

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
````
