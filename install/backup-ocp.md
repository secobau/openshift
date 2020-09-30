### How to backup your Openshift Container Platform

You need to run the following script from a virtual machine INSIDE the VPC network of the OCP cluster:
```
for master in $( oc get node | awk /master/'{ print $1 }' )
do
  ssh core@$master 'mkdir backup; sudo -E /usr/local/bin/cluster-backup.sh backup'
done


```
