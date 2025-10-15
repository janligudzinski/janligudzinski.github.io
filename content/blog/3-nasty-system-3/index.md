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

- once a day, at a parameterizable hour (03:15 Warsaw time - even I'm probably not doing anything then), it runs `pg_dump` for every logical database on our Postgres server that we care to track
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

## Separate deploys for prod and demo

Moving on, let's divorce `prod` from `demo` in our CI/CD pipeline - a push to `develop` should make a different tag in our registry than a push to `master` or `main`, and in fact we already kind of do this, though the `frontend:release-demo` image is still built from the master branch and in sync with it.
