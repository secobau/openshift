1. https://github.com/spring-projects/spring-petclinic
1. https://github.com/dockersamples/dockercoins
1. https://ap-south-1.console.aws.amazon.com/cloud9
   
   In order to deploy petclinic and dockercoins in AWS Cloud9:
   ```bash
   docker swarm init
   
   project=spring-petclinic
   test -f $project.yaml && rm -f $project.yaml
   wget https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/swarm/$project.yaml
   sed --in-place /node/s/worker/manager/ $project.yaml
   docker stack deploy -c $project.yaml $project
   docker service ls

   docker stack rm $project
   test -f $project.yaml && rm -f $project.yaml

   project=dockercoins
   test -f $project.yaml && rm -f $project.yaml
   wget https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/swarm/$project.yaml
   sed --in-place /node/s/worker/manager/ $project.yaml
   docker stack deploy -c $project.yaml $project
   docker service ls

   docker stack rm $project
   test -f $project.yaml && rm -f $project.yaml
   
   
   ``` 
1. https://console-openshift-console.apps.openshift.sebastian-colomar.es
1. https://oauth-openshift.apps.openshift.sebastian-colomar.es/oauth/token/request

   In order to access the Openshift cluster from AWS Cloud9 terminal:
   ```bash
   oc login --token=xxx-yyy --server=https://api.openshift.sebastian-colomar.es:6443
   
   
   ```   
   In order to deploy petclinic and dockercoins in Red Hat Openshift:
   ```bash
   user=dev-x
   
   project=spring-petclinic
   
   oc new-project $project-$user
   oc apply -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc get deployment -n $project-$user
   
   oc delete -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc delete project $project-$user
   
   project=dockercoins
   
   oc new-project $project-$user
   oc apply -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc get deployment -n $project-$user
   
   oc delete -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc delete project $project-$user


   ```
1. Troubleshooting Dockercoins   
1. https://github.com/xwiki-contrib/docker-xwiki
   * https://github.com/secobau/docker-xwiki/tree/openshift

   In order to deploy xwiki in Red Hat Openshift:
   ```bash
   user=dev-x
   
   project=docker-xwiki
   
   oc new-project $project-$user
   oc apply -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc get deployment -n $project-$user
   
   oc delete -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/$project.yaml
   oc delete project $project-$user


   ```
1. https://github.com/secobau/proxy2aws/tree/openshift
   * https://github.com/secobau/nginx
   * https://hub.docker.com/r/secobau/nginx

   1. In order to deploy phpinfo in Red Hat Openshift:
      ```bash
      user=dev-x

      project=proxy2aws

      oc new-project $project-$user
      oc apply -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/openshift/$project.yaml
      oc get deployment -n $project-$user

      oc delete -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/openshift/$project.yaml
      oc delete project $project-$user


      ```
   1. In order to deploy phpinfo in Red Hat Openshift through templates:
      ```bash
      user=dev-x

      project=proxy2aws

      oc new-project $project-$user
      oc process -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/openshift/templates/$project.yaml | oc apply -n $project-$user -f -
      oc get deployment -n $project-$user

      oc process -f https://raw.githubusercontent.com/secobau/$project/openshift/etc/docker/kubernetes/openshift/templates/$project.yaml | oc delete -n $project-$user -f -
      oc delete project $project-$user


      ```
1. https://github.com/secobau/phpinfo

   1. In order to deploy phpinfo in Red Hat Openshift:
      ```bash
      user=dev-x

      project=phpinfo

      oc new-project $project-$user
      oc apply -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/master/etc/docker/kubernetes/openshift/$project.yaml
      oc get deployment -n $project-$user

      oc delete -n $project-$user -f https://raw.githubusercontent.com/secobau/$project/master/etc/docker/kubernetes/openshift/$project.yaml
      oc delete project $project-$user


      ```
   1. In order to deploy phpinfo in Red Hat Openshift through templates:
      ```bash
      user=dev-x

      project=phpinfo

      oc new-project $project-$user
      oc process -f https://raw.githubusercontent.com/secobau/$project/master/etc/docker/kubernetes/openshift/templates/$project.yaml | oc apply -n $project-$user -f -
      oc get deployment -n $project-$user

      oc process -f https://raw.githubusercontent.com/secobau/$project/master/etc/docker/kubernetes/openshift/templates/$project.yaml | oc delete -n $project-$user -f -
      oc delete project $project-$user


      ```
1. Service Mesh:
   1. https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_install/installing-ossm.html
   1. https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_day_two/ossm-example-bookinfo.html
1. https://github.com/kubernetes/kubernetes/issues/77086
   
   ```
   apiVersion: project.openshift.io/v1
   kind: Project
   metadata:
     name: delete-dev-x
   spec:
     finalizers:
     - foregroundDeletion


   ```
   ```
   oc delete project delete-dev-x
   
   
   ```
   ```
   oc get ns delete-dev-x --output json | sed '/ "foregroundDeletion"/d' | curl -k  -H "Authorization: Bearer xxx" -H "Content-Type: application/json" -X PUT --data-binary @- https://api.openshift.sebastian-colomar.es:6443/api/v1/namespaces/delete-dev-x/finalize
   
   
   ```
