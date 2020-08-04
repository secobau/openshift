### How to backup Quay:

```
oc exec -it $( oc get pod --namespace quay-enterprise | awk /quay-postgresql/'{ print $1 }' ) --namespace quay-enterprise -- sh

date=$( date +%F_%H%M%S )
dir=/var/lib/pgsql/data/backup/postgres/
mkdir -p $dir/$date
pg_dump quay > $dir/$date/quay.sql
pg_dumpall > $dir/$date/quay_all.sql
cd $dir
tar -C $dir cf $dir/quay-$date.tar $date
gzip $dir/quay-$date.tar
rm -rf $dir/$date


```
