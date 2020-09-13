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
Creating a VPC in AWS:
```BASH
VpcCidr=10.0.0.0/16
AvailabilityZoneCount=3
SubnetBits=13

file=ocp-vpc.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/VpcCidr_Value/"$VpcCidr"/ $dir/$file
sed --in-place s/AvailabilityZoneCount_Value/"$AvailabilityZoneCount"/ $dir/$file
sed --in-place s/SubnetBits_Value/"$SubnetBits"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json


```
Once the stack creation is completed you can get the following values:
```BASH
PrivateSubnets="$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[0].OutputValue --output text )"
PublicSubnets="$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[1].OutputValue --output text )"
VpcId="$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[2].OutputValue --output text )"

sudo yum install --assumeyes jq

InfrastructureName="$( jq --raw-output .infraID $dir/metadata.json )"
HostedZoneId="$( aws route53 list-hosted-zones-by-name | jq --arg name "$DomainName." --raw-output '.HostedZones | .[] | select(.Name=="\($name)") | .Id' | cut --delimiter / --field 3 )"


```
Creating networking and load balancing components in AWS:
```BASH
file=ocp-route53.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/ClusterName_Value/"$ClusterName"/ $dir/$file
sed --in-place s/HostedZoneId_Value/"$HostedZoneId"/ $dir/$file
sed --in-place s/HostedZoneName_Value/"$DomainName"/ $dir/$file
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/Publish_Value/"$Publish"/ $dir/$file
sed --in-place s/PrivateSubnets_Value/"$PrivateSubnets"/ $dir/$file
sed --in-place s/PublicSubnets_Value/"$PublicSubnets"/ $dir/$file
sed --in-place s/VpcId_Value/"$VpcId"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json --capabilities CAPABILITY_NAMED_IAM


```
Once the stack creation is completed you can get the following values:
```BASH
ExternalApiTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[0].OutputValue --output text )
InternalApiTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[1].OutputValue --output text )
RegisterNlbIpTargetsLambdaArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[5].OutputValue --output text )
InternalServiceTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[6].OutputValue --output text )


```
Creating security group and roles in AWS:
```BASH
file=ocp-roles.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/PrivateSubnets_Value/"$PrivateSubnets"/ $dir/$file
sed --in-place s/VpcCidr_Value/"$( echo $VpcCidr | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/VpcId_Value/"$VpcId"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json --capabilities CAPABILITY_NAMED_IAM


```
Once the stack creation is completed you can get the following values:
```BASH
MasterSecurityGroupId=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[0].OutputValue --output text )


```
Creating the bootstrap node in AWS:
```BASH
RhcosAmi=ami-0754b15d212830477
AllowedBootstrapSshCidr=0.0.0.0/0
BootstrapIgnitionLocation=s3://$InfrastructureName/bootstrap.ign
AutoRegisterELB=yes

aws s3 mb s3://$InfrastructureName
aws s3 cp bootstrap.ign $BootstrapIgnitionLocation
aws s3 ls s3://$InfrastructureName/

file=ocp-bootstrap.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/RhcosAmi_Value/"$RhcosAmi"/ $dir/$file
sed --in-place s/AllowedBootstrapSshCidr_Value/"$( echo $AllowedBootstrapSshCidr | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/PublicSubnet_Value/"$( echo $PublicSubnets | cut --delimiter , --field 1 )"/ $dir/$file
sed --in-place s/MasterSecurityGroupId_Value/"$MasterSecurityGroupId"/ $dir/$file
sed --in-place s/VpcId_Value/"$VpcId"/ $dir/$file
sed --in-place s/BootstrapIgnitionLocation_Value/"$BootstrapIgnitionLocation"/ $dir/$file
sed --in-place s/AutoRegisterELB_Value/"$AutoRegisterELB"/ $dir/$file
sed --in-place s/RegisterNlbIpTargetsLambdaArn_Value/"$RegisterNlbIpTargetsLambdaArn"/ $dir/$file
sed --in-place s/ExternalApiTargetGroupArn_Value/"$ExternalApiTargetGroupArn"/ $dir/$file
sed --in-place s/InternalApiTargetGroupArn_Value/"$InternalApiTargetGroupArn"/ $dir/$file
sed --in-place s/InternalServiceTargetGroupArn_Value/"$InternalServiceTargetGroupArn"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json



```
