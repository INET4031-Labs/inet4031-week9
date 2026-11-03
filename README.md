# inet4031-week9 (Week 9 Lab Reference Repo)

Week 9 of INET 4031: data observability and backup. Students add a Postgres exporter to the Prometheus they installed in Week 6, so they can watch their database's size and row counts, and they build a scheduled backup of that database with a restore script. They keep a copy of a backup on their own computer so it survives the VM, then run a drill: wipe every table, notice it in Prometheus, restore from the backup, and time the recovery. The exporter goes in through the pipeline from Weeks 7 and 8, and its NetworkPolicy change is the deletion that Week 8's approval step stops, so students read the plan and approve it themselves.

## Where the lab is

The step-by-step instructions are in this repo's [wiki](https://github.com/INET4031-Labs/inet4031-week9/wiki). Students follow the wiki, not this README.

## Nothing to clone this week

As in Weeks 5 to 8, there are no starter files. Students create every file themselves from the wiki, inside the personal repo they have used all semester. They do not clone this repo.

## What students end up with

In their own repo, committed through pull requests:

```
week-5/
  exporter.tf     (new: the Postgres exporter Deployment and its Service)
  network.tf      (replaced: a renamed NetworkPolicy that admits web and the exporter)
week-9/
  sql.sh          (run SQL in the database pod, or count rows in every table)
  backup.sh       (dump, check, keep the newest 12)
  restore.sh      (restore a dump into the running database)
  check.sh        (PASS or FAIL for the whole setup)
  README.md       (what this does, a runbook with measured numbers, how to rebuild)
```

Outside the repo:

```
the VM          ~/backups/db-<timestamp>.dump   (mode 600, in a mode 700 folder)
                ~/backup.log and ~/.backup.lock
                a cron line: */5 * * * * <repo>/week-9/backup.sh >> ~/backup.log 2>&1
the cluster     a db-exporter Deployment and Service, and a db-clients NetworkPolicy
their computer  a copy of one backup, pulled by hand with scp
```

No new passwords or keys are created, and `.gitignore` does not change. The exporter reuses the ConfigMap and Secret from Week 5. Backups must never be committed, because the repo is public and a dump holds the database's contents. The wiki says so, and the "should not exist" list includes any `.dump` file.

## Prerequisites

- Week 8 finished, with the ruleset on, the gate running on pull requests, and the new `deploy.sh` with the approval step. The last lines of `~/deploy.log` should say `up to date`.
- The stack running: `web` and `db` `Running`, and the three Prometheus pods `Running` in `monitoring`. The `week-6/pq.sh` script from Week 6 must still exist.
- At least a few tickets in the app. The drill needs data to lose.
- The personal repo from Week 3, still public, with a clean working tree on `master`.
- The SSH port forward from Week 3 (host port 2222) and the VM user's password, for the `scp` step. The VM has to be reachable from the student's own computer.
- The `cron` service running, as in Week 7.
- Internet access from the VM to pull the exporter image from `quay.io`.

## What the lab covers

- The difference between watching a system and watching its data, and what RPO and RTO mean
- Failure domains: why a backup on the same VM as the database shares its fate
- Why a logical backup (`pg_dump`) is used and not a copy of the volume's files, on a running database
- A helper script that runs SQL in the database pod through `kubectl exec`
- A backup script with a lock, a temporary file, a readability check, retention, and a refusal to keep a dump of an empty database
- Scheduling it with cron, and what that means for RPO when the VM is often off
- Copying a backup off the VM by pulling it to the student's own computer, and checking the copy with a SHA-256 hash
- A Postgres exporter added through OpenTofu, scraped by the existing Prometheus because its Service carries `prometheus.io` annotations
- Reading `pg_up`, `pg_database_size_bytes` and `pg_stat_user_tables_n_live_tup` with `pq.sh`
- Reading a plan that deletes something, confirming it names only the old NetworkPolicy, and approving it with `tofu apply`
- A drill: `DROP SCHEMA public CASCADE`, noticing it in Prometheus, restoring from an explicit file, and timing it
- A check script, and a README that doubles as a runbook with the measured numbers

## Things to know before running it

- Written against Postgres 16 (the lab's `postgres:16`), `postgres_exporter` v0.20.1 (`quay.io/prometheuscommunity/postgres-exporter`), and the chart and Prometheus versions from Week 6 (chart 29.35.0). The scan policy is Week 8's six rules, so the exporter has to carry CPU and memory requests and limits, or the pull request's `security scan` fails.
- The restic, MinIO and scheduled GitHub Actions backup from the shared-VM section's version are not used here. These VMs are individual, often off, and eventually deleted, the course is zero-cost, and a GitHub runner cannot reach a VM behind NAT, which Week 7 already covered. The schedule lives in cron on the VM, and the off-VM copy is pulled by the student's own computer.
- The lab never assumes the app's table name. It lists the tables, counts the rows in each one, and wipes the data with `DROP SCHEMA public CASCADE`, which works whatever the app calls its table.
- `backup.sh` runs `pg_dump --format=custom` inside the `db` pod and streams it to a file. It writes to a `.partial` name, reads the dump back with `pg_restore --list`, and only then renames it. A dump with no table data is not kept, so a backup of an empty database can never replace good ones. Cron runs it every five minutes for the lab, and it keeps the newest 12.
- The drill restores from an explicit file, picked before the data is wiped. If cron runs during the outage, it logs `SKIPPED` and keeps the good backups. If the app recreates an empty table first, a later backup would be valid but empty, which is what Q8 asks about.
- `restore.sh` uses `pg_restore --clean --if-exists --no-owner --exit-on-error`. In testing it worked when the tables were missing, when an empty table had been recreated, and when the data was already there, with no duplicate rows.
- The exporter reaches Postgres through the `db` Service with `DATA_SOURCE_URI`, `DATA_SOURCE_USER` and `DATA_SOURCE_PASS`. The database name is filled in by Kubernetes with `$(POSTGRES_DB)` from a variable defined just above it in the same container. It connects as the database superuser, because that is the only account the lab has. A read-only monitoring role is the better answer, and it is listed as a known limitation.
- The wiki renames the NetworkPolicy on purpose. `db-from-web-only` would be wrong once the exporter is admitted, and renaming makes OpenTofu delete the old one and create a new one. The Week 8 deploy script counts the deletion and stops with `BLOCKED`, and the student approves it with `tofu apply` after checking that the plan names only the policy. The expected plan is `3 to add, 0 to change, 1 to destroy`.
- The metric `pg_stat_user_tables_n_live_tup` is Postgres's own running estimate of live rows. It can differ slightly from an exact `count(*)`. After the wipe its series disappears while `pg_up` stays `1`, and it returns after the restore. In testing this showed up in Prometheus within one scrape.
- Prometheus scrapes once a minute and starts with no history, so the first answers after the deploy can take a minute or two, and "did monitoring notice" in the drill takes about a minute. That time counts toward the recovery time, as it would in an outage.
- Alertmanager and the pushgateway stayed off in Week 6, so nothing alerts when `pg_up` drops or when backups go stale. Q7 points at the pushgateway as the way to publish a backup-age metric.
- Run the whole drill in one terminal. The timer and the measurements are shell variables.
- Cron and the backups are on the VM, not in the repo, so a new VM starts without them. The student's README records how to rebuild.

## Common failure points

- **`./week-9/sql.sh` prints `(no tables)` at the start.** The app has not created its table yet. Open the app once and add a ticket.
- **`backup.sh` logs `WAITING`.** The cluster or the `db` pod was not ready, which is normal right after the VM boots. The next run tries again.
- **`backup.sh` logs `FAILED`.** `kubectl exec` into the `db` pod failed or the dump failed. Run `kubectl get pods` and `kubectl logs deploy/db --tail=20`. Nothing is kept on failure, and no `.partial` file is left behind.
- **`backup.sh` logs `SKIPPED`.** The dump had no tables. This is the safety check, and it is expected during the drill while the data is wiped.
- **The backup log is empty after six minutes.** Cron is not running, or the line is missing. `crontab -l` and `systemctl is-active cron`. If the backup line appears twice, remove one with `crontab -e`.
- **`scp` fails or hangs.** The VM's SSH port is not forwarded as 2222, the command was run on the VM instead of on the student's own computer, or the VM is off. The VS Code Explorer's Download option is the alternative.
- **The two hashes differ.** The copy was interrupted or an older file was compared. Copy the file again and hash the same file name on both sides.
- **The merge of the exporter pull request ends in `BLOCKED`.** Expected. The student reads the plan, checks the destroyed line names `db_from_web_only`, and runs `tofu apply` in `week-5/`. If the line names `db_data` or a Deployment, stop and find out why.
- **`tofu apply` by hand fails to find variables.** It was run outside `week-5/`, where `terraform.tfvars` lives.
- **The `security scan` fails on the exporter pull request.** The `resources` block is missing or was pasted wrongly. `gh run view --log-failed` shows the rule and file.
- **`db-exporter` is in `ImagePullBackOff`.** The VM cannot reach `quay.io`, or the tag is wrong.
- **`db-exporter` is in `CrashLoopBackOff`, or `pg_up` is `0`.** `kubectl logs deploy/db-exporter --tail=5` shows a connection error. Check that `kubectl get networkpolicy` lists `db-clients` and not `db-from-web-only`, and that the apply in Step 6 finished.
- **`pg_up` says `(no results)` after three minutes.** The exporter pod is not ready, or Prometheus has not discovered the Service. Check `kubectl get pods`, and check that the Service has the two `prometheus.io` annotations.
- **The drill's variables are empty.** A new terminal was opened. Type the values by hand from the notes in the Google Doc.
- **The row count in Prometheus still shows the table after the wipe.** It has not scraped since. Wait another half minute.
- **`check.sh` fails on backup age.** The newest backup is more than 15 minutes old because cron is not running or the VM was off. Run `./week-9/backup.sh` and try again.
- **The README still has `_HERE` in it.** The variables were empty when the `sed` ran. Type the numbers in over the placeholders.
- **A `.dump` file was committed.** It holds the database's contents and the repo is public. Remove it from the branch before the pull request merges, and tell the instructor if it already did.

## Further reading

- PostgreSQL `pg_dump`: https://www.postgresql.org/docs/16/app-pgdump.html
- PostgreSQL `pg_restore`: https://www.postgresql.org/docs/16/app-pgrestore.html
- PostgreSQL backup and restore overview: https://www.postgresql.org/docs/16/backup.html
- `postgres_exporter`: https://github.com/prometheus-community/postgres_exporter
- Prometheus, writing exporters: https://prometheus.io/docs/instrumenting/writing_exporters/
- Prometheus pushgateway, for batch jobs: https://prometheus.io/docs/practices/pushing/
- OpenTofu: https://opentofu.org/docs/
