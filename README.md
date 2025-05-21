# postgres-pgvecto-rs

Basic postgres chart that I build for usage with the [Immich Chart](https://github.com/immich-app/immich-charts/blob/main/README.md).
Which requires pgvecto-rs/pgvector.
Most charts fail to use the official docker image form tensorchord out of the box.

The data AND backups are all stored in a single PV for now, so make sure it's a backuped folder!

```bash
# install
CHART=https://github.com/valentinsavenko/postgres-pgvecto-rs/raw/refs/heads/main/postgres-pgvecto-rs-0.1.1.tgz

helm install psql-ps ${CHART} -f values.yaml
```

# database backups
the chart creates a cronjob that runs weekly and creates 10 weeks of backups
```bash
# trigger backup NOW
kubectl create job --from=cronjob/postgres-backup postgres-backup-test

# render restore job template with your conf
helm template psql-ps  ${CHART} --set restore.enabled=true -f values.yaml --show-only templates/job.yaml >> db-job-restore-backup.yaml

# restore most recent bkp
kubectl apply -f db-job-restore-backup.yaml
```
adjust the jobs command script if you want a hardcoded backup-foo.sql to be used instead!