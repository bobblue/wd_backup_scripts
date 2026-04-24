# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Bash-based backup, restore, and tenant-migration scripts for **IBM Watson Discovery on Cloud Pak for Data (CP4D)**. Scripts run on an operator's workstation and drive an OpenShift cluster via `oc`. The current script version lives in [version.txt](version.txt) and is checked against the detected WD version at runtime by `validate_version`.

There is no build system, no test suite, and no package manifest — everything is `bash` + `oc`.

## Runtime dependencies

The caller's machine needs: `bash`, `oc` (OpenShift CLI and an active session), `jq`, `tar`, `gzip`, `curl`. The `mc` (MinIO client) binary is auto-downloaded into `./tmp/all_backup/` by `get_mc` when needed. [wks2wd.sh](wks2wd.sh) additionally requires Node.js ≥ 14 and `zip`/`unzip`. Scripts include `Darwin`/`Linux` branches in [lib/function.bash](lib/function.bash) (`get_stat_command`, `get_sed_reg_opt`, `get_base64_opt`) so both macOS and Linux are supported.

## Common commands

```bash
# Full backup / restore (orchestrator — this is the usual entry point)
./all-backup-restore.sh backup [-n NAMESPACE] [-t TENANT]
./all-backup-restore.sh restore -f watson-discovery_YYYYMMDD_HHMMSS.backup [-m mapping.txt]

# Resume after a failure mid-run; values: wddata|etcd|postgresql|elastic|minio|archive|migration|post-restore
./all-backup-restore.sh restore -f FILE --continue-from elastic

# Skip components to produce a partial (INCOMPLETE) backup/restore — caller must recover the
# skipped state manually (e.g. reindex Elastic from PG+MinIO). Must be used symmetrically on both
# backup and restore so the orchestrator doesn't look for a missing tmp/<component>.backup.
./all-backup-restore.sh backup  -n NS --skip-components elastic
./all-backup-restore.sh restore -f FILE --skip-components elastic

# Back up only the Elasticsearch schema (mappings+settings), not documents. Restore leaves the
# cluster operational; collections become searchable again after re-ingesting from source.
./all-backup-restore.sh backup --elastic-schema-only

# Escape hatch for WD 4.7.0 <= ver < 5.2.0 when the WD operator can't reconcile
# 'wd.spec.elasticsearch.sharedStoragePvc' (e.g. CR stuck in Updating). Caller must manually
# attach the PVC (volume + volumeMount at /workdir/shared_storage on the 'elasticsearch'
# container) to the elastic data StatefulSet before running, and detach it after.
./all-backup-restore.sh backup --elastic-shared-pvc my-rwx-pvc --elastic-pvc-preattached

# Run a single component against a tenant (same shape for wddata/etcd/postgresql/elastic/minio)
./elastic-backup-restore.sh backup  TENANT -f elastic.backup
./elastic-backup-restore.sh restore TENANT -f elastic.backup

# Tenant-model migrations (run implicitly by all-backup-restore.sh during restore when needed)
./st-mt-migration.sh TENANT            # single-tenant -> multi-tenant (pre-4.0.6 backup into newer WD)
./mt-mt-migration.sh -i TENANT         # remap tenant IDs between MT deployments

# Reindex before backup if the cluster still has ES6/ES7 indices (also available via --reindex-old-index)
./reindex_es6_indices.sh
./reindex_es7_indices.sh

# Ops/diagnostics
./post-restore.sh TENANT                        # runs automatically at end of restore
./run-migrator.sh TENANT                        # launches the wd-migrator job
./run-postgres-config-job.sh TENANT             # re-applies the configure-postgres job
./mt-migration-size-estimator.sh -n NS          # estimates PG storage headroom needed during MT migration
./openshiftCollector_4.0.0.sh                   # dumps OpenShift diag bundle
./wks2wd.sh --inspect types.json corpus.zip     # WKS type converter (Node.js)
```

Key environment variables (in addition to `--flags`): `OC_ARGS`, `WD_VERSION` (skip auto-detection), `BACKUP_RESTORE_LOG_LEVEL` (`ERROR|WARN|INFO|DEBUG`), `BACKUP_RESTORE_LOG_DIR`, `BACKUP_RESTORE_IN_POD=true` (same as `--use-job`), `TMP_PVC_NAME`, `ELASTIC_SHARED_PVC`, `ELASTIC_PVC_PREATTACHED=true` (mirror of `--elastic-pvc-preattached`), `ELASTIC_OPERATOR_NAMESPACE` / `ELASTIC_OPERATOR_DEPLOY_NAME` (override auto-detection of the ElasticSearch operator deployment when the `olm.owner=ibm-elasticsearch-operator.v*` label doesn't match), `FILE_STORAGE_CLASS`, `SKIP_QUIESCE`, `SKIP_COMPONENTS` (space-separated; mirror of `--skip-components`), `CLEAN`.

## Architecture

### Orchestrator + component-script model

[all-backup-restore.sh](all-backup-restore.sh) is a thin driver. For each entry in `ALL_COMPONENT` (`wddata etcd postgresql elastic minio` on WD ≥ 2.1.3; adds `hdp` on older) it shells out to the matching `<component>-backup-restore.sh` with `command tenant -f ./tmp/<component>.backup`. Each component script is also a standalone entry point with the same CLI shape — that is why they re-source `lib/function.bash` and re-parse `-f`/`-n` themselves. On backup, the orchestrator tars `./tmp/` into a single archive; on restore it extracts first, then runs components, then runs `st-mt-migration.sh` and/or `mt-mt-migration.sh` when `require_st_mt_migration` / `require_mt_mt_migration` return true, then `post-restore.sh`.

Quiesce/unquiesce bracket the whole run (see `quiesce`/`unquiesce` in `lib/function.bash`); `trap_add`/`trap_remove` layer cleanup handlers so a Ctrl-C always unquiesces unless `--quiesce-on-error=true`.

### Version-gated behavior

Virtually every non-trivial code path is gated by `compare_version "$WD_VERSION" "X.Y.Z"`. Examples: MinIO is skipped on WD < 2.1.3 (HDP replaces it); S3-via-`aws` CLI replaces the MinIO `mc` path on WD ≥ 5.2.1 ([minio-backup-restore.sh:68](minio-backup-restore.sh#L68)); Elastic shared-PVC mount is used for 4.7.0 ≤ WD < 5.2.0 ([elastic-backup-restore.sh:41](elastic-backup-restore.sh#L41)); reindex ES6→ES7 vs ES7→OS2 is gated on 2.2.0/4.8.6/5.2.0/5.3.0 boundaries ([all-backup-restore.sh:397-414](all-backup-restore.sh#L397-L414)); Helm-managed vs Ansible-managed install changes probing behavior starting 5.3.0 (`is_managed_by_helm`). When adding or changing behavior, look for the nearest `compare_version` and decide which version ranges your change applies to — there is no single "latest path."

`get_version` detects the running WD version by inspecting the `wd` custom resource, the `wd-management` pod/image, or the management statefulset, with fallbacks down to `2.1`. `validate_version` then refuses to run if the scripts are older than the cluster.

### Shared library: `lib/function.bash`

Nearly all logic lives here (~2000 lines). Conceptual groups:
- **Logging / traps**: `brlog LEVEL MESSAGE`, `trap_add`/`trap_remove`/`disable_trap`.
- **`oc` helpers with retry**: `_oc_cp` (retries + `--retries=` when supported), `run_cmd_in_pod`, `fetch_cmd_result`, `run_script_in_pod`, `kube_cp_from_local`/`kube_cp_to_local` (auto-splits files > `BACKUP_RESTORE_SPLIT_SIZE` = 500 MB and gzip-streams them).
- **Pod discovery / env setup**: `get_elastic_pod`, `get_elastic_pod_container`, `get_primary_pg_pod`, `get_etcd_pod`, `setup_etcd_env`, `setup_pg_env`, `setup_s3_env`, `launch_s3_pod`.
- **Job templating**: `run_pg_job`, `launch_utils_job`, `get_job_pod`, `wait_job_running`. These `sed`-substitute placeholders (`#image#`, `#pg-configmap#`, `#service-account#`, …) into templates from [src/](src/) and `oc apply` them.
- **Archive helpers**: `archive_backup`, `gzip_to_plain_tar`, verify paths (`VERIFY_ARCHIVE`, `VERIFY_BACKUPFILE`, `VERIFY_DATASTORE_ARCHIVE`).
- **Flow control**: `quiesce`/`unquiesce`, `scale_resource`, `restart_job`, `check_datastore_available`, `create_elastic_shared_pvc`.

### `src/` — artifacts copied into pods

Three kinds of files:
1. **`*-in-pod.sh`** (e.g. [src/elastic-backup-restore-in-pod.sh](src/elastic-backup-restore-in-pod.sh), [src/s3-backup-restore-in-pod.sh](src/s3-backup-restore-in-pod.sh)) — copied into a running pod with `oc cp` and executed there, so they can only assume tools present inside that image.
2. **`*-job-template.yml`** — Kubernetes Job manifests with `#placeholder#` tokens that the shell scripts fill via `sed` before `oc apply`.
3. **`*.pgupdate` / `*.etcdupdate` / `*.elupdate` / `*.hdpupdate` / `*.wdupdate` / `*.minioupdate`** — data-store-specific command fragments that [lib/restore-updates.bash](lib/restore-updates.bash) iterates over and executes inside the matching pod during restore. Drop a new file here to inject a one-off restore-time fixup.

### Tenant mapping / multi-tenant migrations

- A backup from a pre-multi-tenant cluster (WD ≤ 4.0.5) restored into a multi-tenant cluster (WD ≥ 4.0.6) triggers `st-mt-migration.sh`, which flips the `watsonDiscoveryEnableMigration` / `watsonDiscoveryMigrationSourceVersion` annotations on the `wd` CR and polls the `watsondiscoverymigration` resource.
- [mt-mt-migration.sh](mt-mt-migration.sh) runs when the backup's instance IDs don't match the target's. It rewrites tenant IDs across PG databases (`dadmin`, `cnm`, `ranker_training`, `sdu`), copies MinIO buckets via [src/minio-mt-migration.sh](src/minio-mt-migration.sh), and reindexes Elasticsearch via [src/elastic-mt-migration.sh](src/elastic-mt-migration.sh). A mapping file (`-m`) passed to `all-backup-restore.sh restore` drives which source tenants map to which destinations; see `create_backup_instance_mappings` / `check_instance_mappings`.

### On-disk layout during a run

All working state is under `./tmp/` (configurable per-component: `tmp/all_backup`, `tmp/elastic_workspace`, `tmp/pg_backup`, …). The orchestrator writes `tmp/version.txt` (`BACKUP_VERSION_FILE`) containing the WD version at backup time — that's how restore decides which migration paths apply (`get_backup_version`). `./tmp_split_backup` holds split chunks during large `oc cp` transfers. Per-component logs land under `wd-backup-restore-logs-YYYYMMDD_HHMMSS/` (override with `--log-output-dir`).

## Conventions to follow when editing

- When adding a new component step, mirror the pattern: standalone-invocable script, `source lib/restore-updates.bash` + `lib/function.bash`, parse `command tenant [-f file] [-n ns]`, set `CURRENT_COMPONENT` for log routing, write output under `tmp/<component>_workspace`, and append the component name to `ALL_COMPONENT` in `all-backup-restore.sh`.
- Gate any new behavior with `compare_version` against the specific WD versions it targets — don't assume "latest." If you introduce a feature, also bump [version.txt](version.txt) so `validate_version` accepts it on matching clusters.
- New version-specific restore fixups go in `src/*.pgupdate` (or `.etcdupdate`/`.elupdate`/etc.), not inline in the component scripts — `lib/restore-updates.bash` picks them up automatically.
- Templates in `src/*.yml` use `#token#` placeholders; keep substitutions `sed`-friendly (no `/` in values that replace into a path) and follow the existing `#image#`, `#pg-configmap#`, `#pg-secret#`, `#release#`, `#namespace#`, `#service-account#` naming.
- Never call `oc cp` directly — use `_oc_cp` (retries) or `kube_cp_from_local`/`kube_cp_to_local` (handles size-splitting and optional gzip streaming via `TRANSFER_WITH_COMPRESSION`).
- Use `brlog LEVEL "msg"` for all user-facing output so log levels and timestamps stay consistent.
