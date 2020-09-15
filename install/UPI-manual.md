First follow these instructions:
* https://github.com/secobau/openshift/blob/master/install/initial.md

After the previous step is finished then you may proceed.

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
sed --in-place s/VpcCidr_Value/"$( echo $VpcCidr | sed 's/\//\\\//g' )"/ $dir/$file
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
file=ocp-route53-$Publish.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/ClusterName_Value/"$ClusterName"/ $dir/$file
sed --in-place s/HostedZoneId_Value/"$HostedZoneId"/ $dir/$file
sed --in-place s/HostedZoneName_Value/"$DomainName"/ $dir/$file
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/Publish_Value/"$Publish"/ $dir/$file
sed --in-place s/PrivateSubnets_Value/"$PrivateSubnets"/ $dir/$file
sed --in-place s/VpcId_Value/"$VpcId"/ $dir/$file

test $Publish = External && sed --in-place s/PublicSubnets_Value/"$PublicSubnets"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json --capabilities CAPABILITY_NAMED_IAM


```
Once the stack creation is completed you can get the following values:
```BASH
InternalApiTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[1].OutputValue --output text )
PrivateHostedZoneId=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[3].OutputValue --output text )
RegisterNlbIpTargetsLambdaArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[5].OutputValue --output text )
InternalServiceTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[6].OutputValue --output text )

test $Publish = External && ExternalApiTargetGroupArn=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[0].OutputValue --output text )


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
MasterInstanceProfileName=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[1].OutputValue --output text )
WorkerSecurityGroupId=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[2].OutputValue --output text )
WorkerInstanceProfileName=$( aws cloudformation describe-stacks --stack-name ${file%.yaml} --query Stacks[].Outputs[3].OutputValue --output text )


```
Creating the bootstrap node in AWS:
```BASH
RhcosAmi=ami-0754b15d212830477
AllowedBootstrapSshCidr=0.0.0.0/0
BootstrapIgnitionLocation=s3://$InfrastructureName/bootstrap.ign
AutoRegisterELB=yes
PublicSubnet=$( echo $PublicSubnets | cut --delimiter , --field 1 )

aws s3 mb s3://$InfrastructureName
aws s3 cp $dir/bootstrap.ign $BootstrapIgnitionLocation
aws s3 ls s3://$InfrastructureName/

file=ocp-bootstrap-$Publish.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/RhcosAmi_Value/"$RhcosAmi"/ $dir/$file
sed --in-place s/AllowedBootstrapSshCidr_Value/"$( echo $AllowedBootstrapSshCidr | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/MasterSecurityGroupId_Value/"$MasterSecurityGroupId"/ $dir/$file
sed --in-place s/VpcId_Value/"$VpcId"/ $dir/$file
sed --in-place s/BootstrapIgnitionLocation_Value/"$( echo $BootstrapIgnitionLocation | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/AutoRegisterELB_Value/"$AutoRegisterELB"/ $dir/$file
sed --in-place s/RegisterNlbIpTargetsLambdaArn_Value/"$( echo $RegisterNlbIpTargetsLambdaArn | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/InternalApiTargetGroupArn_Value/"$( echo $InternalApiTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/InternalServiceTargetGroupArn_Value/"$( echo $InternalServiceTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/PublicSubnet_Value/"$PublicSubnet"/ $dir/$file

test $Publish = External && sed --in-place s/ExternalApiTargetGroupArn_Value/"$( echo $ExternalApiTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json --capabilities CAPABILITY_NAMED_IAM


```
Creating the control plane machines in AWS:
```BASH
AutoRegisterDNS=yes
PrivateHostedZoneName=$ClusterName.$DomainName
Master0Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 1 )
Master1Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 2 )
Master2Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 3 )
IgnitionLocation=https://api-int.$PrivateHostedZoneName:22623/config/master
CertificateAuthorities=$( jq .ignition.security.tls.certificateAuthorities[0].source --raw-output $dir/master.ign )
MasterInstanceType=t3a.xlarge

file=ocp-master-$Publish.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/RhcosAmi_Value/"$RhcosAmi"/ $dir/$file
sed --in-place s/AutoRegisterDNS_Value/"$AutoRegisterDNS"/ $dir/$file
sed --in-place s/PrivateHostedZoneId_Value/"$PrivateHostedZoneId"/ $dir/$file
sed --in-place s/PrivateHostedZoneName_Value/"$PrivateHostedZoneName"/ $dir/$file
sed --in-place s/Master0Subnet_Value/"$Master0Subnet"/ $dir/$file
sed --in-place s/Master1Subnet_Value/"$Master1Subnet"/ $dir/$file
sed --in-place s/Master2Subnet_Value/"$Master2Subnet"/ $dir/$file
sed --in-place s/MasterSecurityGroupId_Value/"$MasterSecurityGroupId"/ $dir/$file
sed --in-place s/IgnitionLocation_Value/"$( echo $IgnitionLocation | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/CertificateAuthorities_Value/"$( echo $CertificateAuthorities | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/MasterInstanceProfileName_Value/"$MasterInstanceProfileName"/ $dir/$file
sed --in-place s/MasterInstanceType_Value/"$MasterInstanceType"/ $dir/$file
sed --in-place s/AutoRegisterELB_Value/"$AutoRegisterELB"/ $dir/$file
sed --in-place s/RegisterNlbIpTargetsLambdaArn_Value/"$( echo $RegisterNlbIpTargetsLambdaArn | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/InternalApiTargetGroupArn_Value/"$( echo $InternalApiTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/InternalServiceTargetGroupArn_Value/"$( echo $InternalServiceTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file

test $Publish = External && sed --in-place s/ExternalApiTargetGroupArn_Value/"$( echo $ExternalApiTargetGroupArn | sed 's/\//\\\//g' )"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json


```
Initializing the bootstrap node on AWS with user-provisioned infrastructure:
```BASH
openshift-install-$version wait-for bootstrap-complete --dir $dir --log-level debug


```
Creating the worker nodes in AWS:
```BASH
IgnitionLocation=https://api-int.$PrivateHostedZoneName:22623/config/worker
CertificateAuthorities=$( jq .ignition.security.tls.certificateAuthorities[0].source --raw-output $dir/worker.ign )
WorkerInstanceType=t3a.large
Worker0Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 1 )
Worker1Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 2 )
Worker2Subnet=$( echo $PrivateSubnets | cut --delimiter , --field 3 )

file=ocp-worker.json
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
sed --in-place s/InfrastructureName_Value/"$InfrastructureName"/ $dir/$file
sed --in-place s/RhcosAmi_Value/"$RhcosAmi"/ $dir/$file
sed --in-place s/WorkerSecurityGroupId_Value/"$WorkerSecurityGroupId"/ $dir/$file
sed --in-place s/IgnitionLocation_Value/"$( echo $IgnitionLocation | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/CertificateAuthorities_Value/"$( echo $CertificateAuthorities | sed 's/\//\\\//g' )"/ $dir/$file
sed --in-place s/WorkerInstanceProfileName_Value/"$WorkerInstanceProfileName"/ $dir/$file
sed --in-place s/WorkerInstanceType_Value/"$WorkerInstanceType"/ $dir/$file
sed --in-place s/Subnet0_Value/"$Worker0Subnet"/ $dir/$file
sed --in-place s/Subnet1_Value/"$Worker1Subnet"/ $dir/$file
sed --in-place s/Subnet2_Value/"$Worker2Subnet"/ $dir/$file

file=${file%.json}.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/install/$file --directory-prefix $dir
aws cloudformation create-stack --stack-name ${file%.yaml} --template-body file://$dir/$file --parameters file://$dir/${file%.yaml}.json


```
Logging in to the cluster:
```BASH
export KUBECONFIG=$dir/auth/kubeconfig


```
Approving the CSRs for your machines:
```bash
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve


```
Deleting the bootstrap resources:
```bash
aws cloudformation delete-stack --stack-name ocp-bootstrap


```
Creating the Ingress DNS Records:
```bash
routes="$( oc get --all-namespaces -o jsonpath='{range .items[*]}{range .status.ingress[*]}{.host}{"\n"}{end}{end}' routes | cut --delimiter . --field 1 )"
hostname=$( oc -n openshift-ingress get service router-default -o custom-columns=:.status.loadBalancer.ingress[].hostname --no-headers )
CanonicalHostedZoneNameID=$( aws elb describe-load-balancers | jq -r '.LoadBalancerDescriptions[] | select(.DNSName == "'$hostname'").CanonicalHostedZoneNameID' )
PrivateHostedZoneId="$( aws route53 list-hosted-zones-by-name | jq --arg name "$ClusterName.$DomainName." --raw-output '.HostedZones | .[] | select(.Name=="\($name)") | .Id' | cut --delimiter / --field 3 )"

test $Publish = External && PublicHostedZoneId="$( aws route53 list-hosted-zones-by-name | jq --arg name "$DomainName." --raw-output '.HostedZones | .[] | select(.Name=="\($name)") | .Id' | cut --delimiter / --field 3 )"

for route in $routes
do
aws route53 change-resource-record-sets --hosted-zone-id "$PrivateHostedZoneId" --change-batch '{ "Changes": [ { "Action": "CREATE", "ResourceRecordSet": { "Name": "'$route'.apps.'$ClusterName.$DomainName'", "Type": "A", "AliasTarget":{ "HostedZoneId": "'$CanonicalHostedZoneNameID'", "DNSName": "'$hostname'.", "EvaluateTargetHealth": false } } } ] }'
test $Publish = External && aws route53 change-resource-record-sets --hosted-zone-id "$PublicHostedZoneId" --change-batch '{ "Changes": [ { "Action": "CREATE", "ResourceRecordSet": { "Name": "'$route'.apps.'$ClusterName.$DomainName'", "Type": "A", "AliasTarget":{ "HostedZoneId": "'$CanonicalHostedZoneNameID'", "DNSName": "'$hostname'.", "EvaluateTargetHealth": false } } } ] }'
done


```
Completing an AWS installation on user-provisioned infrastructure:
```bash
openshift-install-$version wait-for install-complete --dir $dir --log-level debug


```
#### Credits

This documentation is based on this: https://docs.openshift.com/container-platform/4.5/installing/installing_aws/installing-aws-user-infra.html
