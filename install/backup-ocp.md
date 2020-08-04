### How to backup your Openshift Container Platform:

```
for master in $( oc get node | awk /master/'{ print $1 }' )
do
  ssh core@$master 'sudo -E /usr/local/bin/cluster-backup.sh backup'
done


```
