# syntax=docker/dockerfile:1
# Bakes a production database dump into DDEV's stock database server image.
#
# The build imports the dump and writes /mysqlbase/custom/base_db.zst —
# the seed location DDEV reserves for derived images, which its entrypoint
# prefers over the stock starter archive /mysqlbase/base_db.zst when
# restoring into an empty datadir on first start. The datadir itself stays
# empty, so every consumer loads the data the same way:
#
# - DDEV's db service: `dbimage:` in .ddev/config.yaml points DDEV at
#   this image, and the stock entrypoint restores the seed into the
#   "database" volume whenever that volume is empty.
# - Standalone `docker run` (e.g. a GitHub Actions service container):
#   the entrypoint restores it into the container's datadir on startup.
#
# The builder stage always runs natively on the build host
# ($BUILDPLATFORM); its product, base_db.zst, is architecture-independent
# InnoDB page data, so one native build feeds every target platform and
# `docker buildx build --platform linux/amd64,linux/arm64` needs no
# per-architecture runners and no emulation.

# The tag should match what the team's DDEV expects (the `db` row of
# `ddev version`); `ddev start` warns when a pinned dbimage was built
# from any other tag. database.yml overrides this via --build-arg from
# the DB_BASE_IMAGE Actions variable; keep its fallback in sync with the
# default below.
ARG BASE_IMAGE=ddev/ddev-dbserver-mariadb-11.8:v1.25.4

FROM --platform=$BUILDPLATFORM ${BASE_IMAGE} AS builder

RUN --mount=type=bind,source=dump.sql.gz,target=/tmp/dump.sql.gz <<'EOT'
set -eux -o pipefail
socket=/var/tmp/mysql.sock

# Restore the stock base database into the (empty) datadir, mirroring the
# restore /docker-entrypoint.sh performs on first boot, including its
# choice of backup tool (MySQL bases ship xtrabackup, MariaDB mariabackup).
backup=mariabackup
stream=mbstream
if command -v xtrabackup >/dev/null; then backup=xtrabackup; stream=xbstream; fi
mkdir -p /var/tmp/base_db
cd /var/tmp/base_db
zstdmt -dc --quiet /mysqlbase/base_db.zst | "${stream}" -x
"${backup}" --prepare --skip-innodb-use-native-aio --target-dir=/var/tmp/base_db
"${backup}" --datadir=/var/lib/mysql --copy-back --skip-innodb-use-native-aio --target-dir=/var/tmp/base_db

# Start a build-time server (socket only) and wait for it; the post-loop
# ping is the timeout backstop.
mysqld --user=root --skip-networking --socket="${socket}" &
pid=$!
for _ in $(seq 120); do
  mysqladmin ping -uroot -proot --socket="${socket}" >/dev/null 2>&1 && break
  kill -s 0 "${pid}"
  sleep 1
done
mysqladmin ping -uroot -proot --socket="${socket}"

# Import into the standard DDEV database.
gunzip -c /tmp/dump.sql.gz | mysql -uroot -proot --socket="${socket}" db

# Write a base backup containing the imported data, in the same format as
# the starter archive (zstd-compressed backup stream), to the derived-image
# seed path the entrypoint prefers over the stock /mysqlbase/base_db.zst.
mkdir -p /mysqlbase/custom
"${backup}" --backup --stream="${stream}" --user=root --password=root --socket="${socket}" | zstdmt --quiet > /mysqlbase/custom/base_db.zst

mysqladmin shutdown -uroot -proot --socket="${socket}"
wait "${pid}"
EOT

# Final stage, per target platform: pure COPY/metadata, executes nothing.
# The datadir keeps its stock empty state, so the entrypoint performs the
# base_db.zst restore on first start.
FROM ${BASE_IMAGE}

COPY --from=builder /mysqlbase/custom/base_db.zst /mysqlbase/custom/base_db.zst

# No USER is declared: DDEV builds a derived image on top of this one and
# runs groupadd/useradd in it without switching to root first, so a
# non-root USER here breaks `ddev start`. The server still refuses to run
# as root: DDEV's compose sets the host user's uid explicitly, and other
# consumers (e.g. GHA service containers) should pass `--user mysql`.

# The inherited healthcheck tolerates only ~30s of initialization before
# reporting unhealthy. Restoring the production data on first start takes
# longer, so allow a generous start period (failures during it don't count
# against the retry limit; success ends it immediately).
HEALTHCHECK --interval=1s --timeout=70s --retries=70 --start-period=10m --start-interval=1s CMD ["/healthcheck.sh"]
