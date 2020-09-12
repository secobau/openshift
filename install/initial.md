In order to install a new Red Hat Openshift cluster in AWS please follow these steps in the precise order.

You will need the Pull Secret generated in this page:
* https://cloud.redhat.com/openshift/install

All the steps will be performed from an AWS Cloud9 terminal with enough privileges (AdministratorAccess will work).

You will need to obtain a valid public domain name before installing the cluster:
* https://console.aws.amazon.com/route53/home

Disable AWS managed temporary credentials in AWS Cloud9 settings. Now you need to create a new Access Key in your AWS IAM Security Credentials and then configure your AWS Cloud9 terminal:
```bash
aws configure


```

Now generate an SSH key pair and add it to the SSH agent. This will allow you to access the cluster nodes through SSH.
```bash
ssh-keygen

eval "$(ssh-agent -s)"
ssh-add $HOME/.ssh/id_rsa


```
Choose a version number:
```bash
version=4.5.8


```
Afterwards you can proceed to install the client and the installer binaries:
```bash
for mode in client install
do
  wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$version/openshift-$mode-linux-$version.tar.gz
  gunzip openshift-$mode-linux-$version.tar.gz
  tar xf openshift-$mode-linux-$version.tar
  rm openshift-$mode-linux-$version.tar
done
mkdir --parents $HOME/bin
for binary in kubectl oc
do
  mv $binary $HOME/bin
done
mv openshift-install $HOME/bin/openshift-install-$version


```
Now you introduce your choice for the name and domain of the cluster:
```bash
export ClusterName=openshift
export DomainName=sebastian-colomar.es


```
Create a directory to place all the configuration files:
```bash
dir="$HOME/environment/openshift/install/$ClusterName.$DomainName"
test -d $dir || mkdir --parents $dir


```
Now you create a configuration file template to be later modified:
```bash
openshift-install-$version create install-config --dir $dir


```
The following script will modify the EC2 instance type so as to choose the cheapest possible type but big enough to correctly set up the cluster:
```bash
cd $dir
wget https://raw.githubusercontent.com/secobau/openshift/master/install/fix-config.sh
chmod +x fix-config.sh && ./fix-config.sh


```
