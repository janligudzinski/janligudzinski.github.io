+++
authors = ["Jan Ligudziński"]
title = "In which I fix a nasty production system: #3 - Backend refactoring, part 1"
description = "Cargo-culting C# projects from work was a bad idea."
date = 2025-10-14
[taxonomies]
tags = ["nasty system", "programming", "rust", "war story", "refactoring"]
+++

## Where we left off

In the [previous post](/blog/2-nasty-system-2) we finished building a complete CI and CD pipeline for our project - things get built before they are merged into master, and when they are they not only get built but also pushed to production, with a Github "runner" on our server that connects to Github and waits for orders to reinstall things.
Two more things we can do before we continue with refactoring the system:
- let's have our `demo` environment run builds from the `develop` branch where we will put the newest features - the demo environment, in our two-man practice, is as much for (rare) demos before clients as it is for running things past the CEO (also rarely). I don't think a dedicated `dev` environment, subdomain, and so on, are justified quite yet - if I'm actively developing something, it stays on my machine anyway.
- let's have our `db` automatically back itself up - let's say to a file on the server - once a day.

## DB backups

I asked GPT to generate me a periodic job to run in the same Compose file as the DB to back it up to files mounted on the machine. What it gave me I then manually edited to remove excess parameterization, reuse the env file we already pass to the database itself, and be its own Dockerfile and shell scripts instead of a disgusting inline YAML string. What it does is:

- once a day, at a parameterizable hour (around 03:15 Warsaw time - even I'm probably not doing anything then), it runs `pg_dump` for every logical database on our Postgres server that we care to track
- this is done with cron calling a separate backup script
- which is also called once at container startup, so we know it's working right away
- the backups are stored in a mounted volume on the host machine, in a directory we can easily access
- the backups roll over after a week, which I think is plenty of time. Considering that our production DB's dump weighs about 600KB at the time of writing these words at the time of operation, we could go absolutely insane and store a month's worth or capture the backups more often, but I don't see the need for it.

Behold:

```yaml
db:
  image: postgres:15-alpine
  networks:
    infra_net:
      aliases: [db]
  env_file: /opt/element/env/db/.env
  ports:
    - "5432:5432" # we definitely want host access (through an SSH tunnel ofc)
  volumes:
    - db_data:/var/lib/postgresql/data
    - /opt/element/env/db/initdb:/docker-entrypoint-initdb.d # this will only run when the volume is empty, btw
db_backup:
  build:
    context: ./db_backup
    dockerfile: Dockerfile
  networks: [infra_net]
  environment:
    DBS: "goldenhand goldenhand_demo"
    RUN_AT: "03:17"
    TZ: "Europe/Warsaw"
  env_file: /opt/element/env/db/.env
  volumes:
    - /opt/element/db_backups:/app/backups
```

```Dockerfile
FROM alpine:3

RUN apk add --no-cache postgresql-client tzdata bash coreutils findutils >/dev/null
COPY backup.sh /app/backup.sh
COPY run.sh /app/run.sh
RUN chmod +x /app/run.sh
RUN chmod +x /app/backup.sh

WORKDIR /app
ENTRYPOINT ["/app/run.sh"]
```

```bash
# run.sh:
#!/usr/bin/env bash
set -euo pipefail

: "${RUN_AT:?Set RUN_AT like HH:MM}"
: "${DBS:?Set DBS (space-separated database names)}"

# Map to PG* if not set
: "${PGUSER:=${POSTGRES_USER:-}}"
: "${PGPASSWORD:=${POSTGRES_PASSWORD:-}}"
: "${PGHOST:=db}"
: "${PGPORT:=5432}"
export PGPASSWORD
mkdir -p /app/backups

# Build a minimal crontab entry
H="${RUN_AT%:*}"
M="${RUN_AT#*:}"

# Write crontab (to user’s crontab path on Alpine)
CRON_FILE="${HOME}/crontab"
CRON_ENV="DBS='${DBS}' PGHOST='${PGHOST}' PGPORT='${PGPORT}' PGUSER='${PGUSER}' PGPASSWORD='${PGPASSWORD}'"
echo "${M} ${H} * * * ${CRON_ENV} /app/backup.sh >> ${HOME}/cron.log 2>&1" > "$CRON_FILE"

# Install crontab
crontab "$CRON_FILE"

echo "[pgbackup] Running initial backup now..."
/app/backup.sh || true
echo "[pgbackup] Scheduled daily at ${RUN_AT} ${TZ:-UTC}"

# Run cron in foreground so the container stays up
# busybox crond is already present; -f stays in foreground, -l 8 is verbose
touch /var/log/cron.log
exec crond -f -l 8

# backup.sh:
#!/usr/bin/env bash
set -euo pipefail
: "${DBS:?missing DBS}"

: "${PGUSER:=${POSTGRES_USER}}"
: "${PGPASSWORD:=${POSTGRES_PASSWORD}}"
: "${PGHOST:=db}"
: "${PGPORT:=5432}"
export PGPASSWORD

# wait for DB to be ready (check the first DB name)
first_db="${DBS%% *}"
until pg_isready -h "${PGHOST:-db}" -p "${PGPORT:-5432}" -U "${PGUSER:-postgres}" -d "$first_db" >/dev/null 2>&1; do
  echo "[backup] waiting for database..."
  sleep 2
done

ts="$(date +%F_%H-%M-%S)"
failures=0

# robust split
IFS=' ' read -r -a dbs <<< "$DBS"

for d in "${dbs[@]}"; do
  [ -z "$d" ] && continue
  out="/app/backups/${d}_${ts}.sql.gz"
  echo "[backup] dumping ${d} -> ${out}"
  if ! pg_dump -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" -d "$d" --no-owner --no-privileges | gzip -9 > "$out"; then
    echo "[backup][ERROR] pg_dump failed for ${d}" >&2
    failures=1
  fi
done

echo "[backup] pruning files older than 7 days"
find /app/backups -type f -name "*.sql.gz" -mtime +6 -print -delete || true

if [ "$failures" -ne 0 ]; then
  echo "[backup] finished with errors" >&2
  exit 1
fi
echo "[backup] done"

```

While this approach is not perfect - we still don't exfiltrate the dumps outside of our machine right away - it's better than what we had up to now, that is to say, *nothing* but my due diligence. And that only occasionally.

>*💀💀💀*

*Yeah*. Now that I've slept on this, considering that we have an S3-clone bucket bought already, it might be a good idea to upload the dumps there as well in case the machine ever goes poof. It's also not best practice and probably not legal to get it out unencrypted (or even store it unencrypted at rest), so we might have to use `age` after all - generate a key pair on our dev box, then put the public, encrypt-only key on the server (we can even commit it to the repo). Pipe the dumps through it before saving and uploading, if we need to get them out then I can decrypt locally with the private key I have; maybe even print it out on paper in case it's *my* computer that buys the farm.

### Adding encryption

Let's install `age` on my computer with `yay -Syu age` and also add it to the backup container's `run apk add ...` line, then generate the key:

```
~/Dokumenty/Klucze$ age-keygen -o db_backup_key.txt
age-keygen: warning: writing secret key to a world-readable file [non-issue, I don't let other machines or people onto my own computer]
Public key: age1sae4n3twedp6dfxls4fu436z5h0cgwkzrgv2qvj4j6fexvdzvsfq9z47h0 [I can just show you this, what are you gonna do? Encrypt files that only I will be able to decrypt?]
```

```yaml
# ...
db_backup:
    build:
      context: ./db_backup
      dockerfile: Dockerfile
    networks: [infra_net]
    environment:
      AGE_PUBLIC_KEY: age1sae4n3twedp6dfxls4fu436z5h0cgwkzrgv2qvj4j6fexvdzvsfq9z47h0 # dw, safe to commit
# ...
```

```bash
#...
: "${AGE_PUBLIC_KEY:?missing AGE_PUBLIC_KEY}"
#...
out="/app/backups/${d}_${ts}.sql.gz"
out_enc="${out}.age"
echo "[backup] dumping ${d} -> ${out_enc}"
if ! pg_dump -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" -d "$d" --no-owner --no-privileges | gzip -9 | age -r ${AGE_PUBLIC_KEY} > ${out_enc}; then
  echo "[backup][ERROR] pg_dump failed for ${d}" >&2
  failures=1
fi
#...
```

Note that we *first* compress the dump, *then* encrypt it - encrypted data looks much more random than a very repetitive SQL file with lots of `INSERT INTO ...` boilerplate or the same IDs cropping up in multiple places, and so is not going to compress well. We also pipe everything up to `out_enc` so at no point do we write the unencrypted dump to disk.

![backups pulled](backups-pulled.png)

All that's left is to delete the plain `.sql.gz` files.

```
root@ubuntu-8gb-fsn1-1:~# rm /opt/element/db_backups/*.sql.gz
root@ubuntu-8gb-fsn1-1:~# ls /opt/element/db_backups
goldenhand_2025-10-16_14-40-41.sql.gz.age  goldenhand_demo_2025-10-16_14-40-41.sql.gz.age
```

Let's now peek into the files locally:
```
head -c 200 goldenhand_2025*.age
age-encryption.org/v1
-> X25519 lzFBWTPW1t7z6yy6u5DRAy/rLp1n+8Ccw7SBniJW3Wg
e8rGY5FMJ8c6LxTUjbbMlzVE4YeVsvSyuk9/3A5MIGc
--- 8eUblsOQoFHrmTO/BSsKVIPBOu02gtLS7igt4WYPs/E
�L����f�s�▒X�~u���l�����mh�>%

cat goldenhand_2025*.age | age --decrypt -i ~/Dokumenty/Klucze/db_backup_key.txt | gzip -d > demo_dump_decrypted.sql

head -c 200 demo_dump_decrypted.sql
--
-- PostgreSQL database dump
--

\restrict dRwWcw4f7HDYqYbqHqg67shjZrwzrFEeZFjrmax50JUM2zm1FqFHE00nSgbbz1k

-- Dumped from database version 15.14
-- Dumped by pg_dump version 17.6

SET statement_tim%
```

Yeah, this works all right.

### Syncing to S3

Finally, let's upload the dumps to our Hetzner S3. I just had to manually create the `element-misc` bucket where we'll throw these backups and maybe other things of a similar nature if there's a need, and we can reuse our existing project-level credentials; finer-grained permissions aren't offered.

We'll add `rclone` to the container, which is like `rsync` but for S3 and not ssh - a CLI tool that syncs the state of a physical machine's directory with the contents of a bucket. Thankfully it's in the Alpine Linux repo, so once again we just add it to the `apk add` line.

The S3 keys, unlike my backup script's public key, actually need to be secret, so I'll pass them in the untracked .env file the backup script already uses as we have done so far. The 4 variables to include are `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_BUCKET`, `S3_REGION` and `S3_ENDPOINT`), with an `RCLONE_` prefix for `rclone` to pick them up automatically as told by the inline `env_auth` flag (except `BUCKET` which has to be passed explicitly, but I'll keep the RCLONE_ for consistency):

```bash
# backup.sh
# ... (exit early if something goes wrong)
echo "[backup] syncing to S3"

rclone sync /app/backups ":s3,env_auth:/${RCLONE_S3_BUCKET}/main_machine_db_backups/"

echo "[backup] done"
```

![backups pushed](backups-pushed.png)

As we've now also established that the backup job does in fact fire at 03:17 when I've likely already gone to sleep and my users definitely have, we can cut out the line of `run.sh` that runs `backup.sh` once regardless of timing at startup as well.

## Separate deploys for prod and demo

Moving on, let's divorce `prod` from `demo` in our CI/CD pipeline - a push to `develop` should make a different tag in our registry than a push to `master` or `main`, and in fact we already kind of do this, though the `frontend:release-demo` image is still built from the master branch and in sync with it. What's more, with our recent switch from Docker Compose v1 to v2, we can indiscriminately `up -d` with only the containers that have actually changed and been pulled anew getting downed first, so all we really need to do is conditionally switch the tag in our registry pushing action and use that tag in `apps.docker-compose.yml`.
