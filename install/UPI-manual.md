First follow these instructions:
* https://github.com/secobau/openshift/blob/master/install/initial.md

After the previous step is finished you may proceed.

Set the number of compute replicas to zero:
```bash
sed --in-place /compute/,/controlPlane/s/\ 3/\ 0/ $dir/install-config.yaml


```
It is a good idea to make a copy of your configuration file:
```bash
cp $dir/install-config.yaml $dir/install-config.yaml.$( date +%F_%H%M )


```
Now you generate the Kubernetes manifests for the cluster:
```BASH
openshift-install-$version create manifests --dir $dir --log-level debug


```
Remove the Kubernetes manifest files that define the control plane machines:
```BASH
rm --force $dir/openshift/99_openshift-cluster-api_master-machines-*.yaml


```
Remove the Kubernetes manifest files that define the worker machines:
```BASH
rm --force $dir/openshift/99_openshift-cluster-api_worker-machineset-*.yaml


```
Prevent Pods from being scheduled on the control plane machines:
```bash
sed --in-place /mastersSchedulable/s/true/false/ $dir/manifests/cluster-scheduler-02-config.yml


```
If you do not want the Ingress Operator to create DNS records on your behalf, remove the privateZone and publicZone sections from the DNS configuration file:
```bash
sed --in-place /privateZone:/,/id:/d $dir/manifests/cluster-dns-02-config.yml


```
Obtain the Ignition config files:
```BASH
openshift-install-$version create ignition-configs --dir $dir --log-level debug


```
You can use the following CloudFormation template to deploy the VPC that you need for your OpenShift Container Platform cluster:
```BASH
file=ocp-vpc.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file


```
You can use the following CloudFormation template to deploy the networking objects and load balancers that you need for your OpenShift Container Platform cluster:
```BASH
sudo yum install --assumeyes jq

InfrastructureName=$( jq --raw-output .infraID $dir/metadata.json )
HostedZoneId=$( aws route53 list-hosted-zones-by-name | jq --arg name "$DomainName." --raw-output '.HostedZones | .[] | select(.Name=="\($name)") | .Id' | cut --delimiter / --field 3 )

file=ocp-parameters.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/ClusterName_Value/$ClusterName/ $dir/$file
sed --in-place s/HostedZoneId_Value/$HostedZoneId/ $dir/$file
sed --in-place s/HostedZoneName_Value/$DomainName/ $dir/$file
sed --in-place s/InfrastructureName_Value/$InfrastructureName/ $dir/$file
sed --in-place s/Publish_Value/$Publish/ $dir/$file

file=ocp-route53.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/ocp-parameters.json --capabilities CAPABILITY_NAMED_IAM


```
