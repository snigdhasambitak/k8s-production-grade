# devopsassignment

## CREATE A KUBERNETES CLUSTER ON GCP (GCP GIVES FREE CREDITS ON SIGNUP SO THOSE SHOULD SUFFICE FOR THIS EXERCISE) ON THEIR VIRTUAL MACHINES (DO NOT USE GKE) OR USE THE CLUSTER CREATED IN THE LEVEL2 TEST. IF POSSIBLE SHARE A SCRIPT / CODE WHICH CAN BE USED TO CREATE THE CLUSTER.

I am using Terraform for creating a kubernetes cluster on AWS Platform.

Step 1: Create the modules for the creation of the eks cluster, master nodes, VPC and providers. Everything can be found in this directory.

Step 2: Initialize the Terraform state:  

$ terraform init

Step 3: Plan the Deployment

$ terraform plan -var 'cluster-name=POP-DEV' -var 'desired-capacity=3' -out POP-DEV-tf

Step 4: We can apply the plan now

$ terraform apply "POP-DEV-tf"

Step 5: Create the KubeConfig file and save it, that will be used to manage the cluster.

$ terraform output kubeconfig

$ terraform output kubeconfig > ${HOME}/.kube/config-POP-DEV-tf

Step 6: Add this new config to the KubeCtl Config list.

$ export KUBECONFIG=${HOME}/.kube/config-POP-DEV-tf:${HOME}/.kube/config
$ echo "export KUBECONFIG=${KUBECONFIG}" >> ${HOME}/.bashrc

Step 7: The terraform state also contains a config-map we can use for our EKS workers. We can view, save and apply the config maps.

#View
$ terraform output config-map

#Save
$ terraform output config-map > /tmp/config-map-aws-auth.yml

#Apply
$ kubectl apply -f /tmp/config-map-aws-auth.yml

Step 8: Confirm that the nodes are available. If available then we have a working cluster as follows:

$ kubectl get nodes                                                                                         ✔  ⚙  6167  20:50:06
NAME                                           STATUS   ROLES    AGE   VERSION
ip-172-xx-xxx-109.eu-west-1.compute.internal   Ready    <none>   19d   v1.12.7
ip-172-xx-xxx-183.eu-west-1.compute.internal   Ready    <none>   19d   v1.12.7
ip-172-xx-xxx-29.eu-west-1.compute.internal    Ready    <none>   19d   v1.12.7

Step 9: In order to delete the resources created for this EKS cluster, run the following commands:

terraform plan -destroy -out POP-DEV-destroy-tf
terraform apply "POP-DEV-destroy-tf"







## SETUP CI SERVER (JENKINS OR TOOL OF YOUR CHOICE) INSIDE THE KUBERNETES CLUSTER AND MAINTAIN THE HIGH AVAILABILITY OF THE JOBS CREATED. Please access the "Jenkins" folder from root for the config files.

Step 1: Create a deployment file for jenkins and we can set up 3 replicas. This ensures 3 instances will be maintained by the Replication Controller in the event of failure at all time.

The file jenkins-deployment.yaml is defining a Deployment as indicated by the kind field.

The container image name is jenkins and version is latest
The list of ports specified within the spec are a list of ports to expose from the container on the Pods IP address.
Jenkins running on (http) port 8080.
The Pod exposes the port 8080 of the jenkins container.

Step 2: Create a Namespace for Jenkins. So that we will have an isolation for the CI/CD environment.

$ kubectl create ns jenkins

Step 3: Apply the deploymnet file that was created at step 1:

$ kubectl create -f jenkins/jenkins-deployment.yaml --namespace=jenkins


Step 4: To validate that creating the deployment was successful you can invoke:

$ kubectl get deployments --namespace=jenkins

NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jenkins   1         1         1            1           3m

We can also see that a single Pod has been created by invoking:

$ kubectl get pods --namespace=jenkins
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-69278131955-31nj71   1/1       Running   0          7m

Step 4: To make Jenkins accessible outside the Kubernetes cluster the Pod needs to be exposed as a Service. With a local deployment this means creating a NodePort service type. A NodePort service type exposes a service on a port on each node in the cluster. It’s then possible to access the service given the Node IP address and the service nodePort. Also, we have mentioned the nodeport as 30000. So you can access the application on port 30000. A simple service is defined in the file : jenkins-service.yaml

Step 5: The file is defining a Service as indicated by the kind field. The Service is of type NodePort. Other options are ClusterIP (only accessible within the cluster) and LoadBalancer. The list of ports specified within the spec are a list of ports exposed by this service. To create the service execute:

$ kubectl create -f jenkins/jenkins-service.yaml --namespace=jenkins

Step 6: To validate that creating the service was successful you can invoke:

$ kubectl get services --namespace=jenkins
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
jenkins      10.0.0.202   <nodes>       8080:30000/TCP   3m

Step 7: Now if you browse to any one of the Node IP on port 30000, you will be able to access the Jenkins dashboard.

http://<node-ip>:3000

Step 8: Inorder to login to jenkins we need the admin password for the admin user. We can get that by accessing the pod logs as follows:

$ kubectl logs jenkins-deployment-7867864836-j00u5 --namespace=jenkins

Jenkins initial setup is required. An admin user has been created and a password created.
Please use the following password to proceed to installation:

8ui6yd7ban8ka72b38vwklsv8w2vh

This may also be found at : /var/jenkins_home/secrets/initialAdminPassword

************************************************************
************************************************************
************************************************************


Step 9: Now we need to create a Horizontal Pod Autoscaler that maintains between 1 and 10 replicas of the Pods controlled by the jenkins deployment we created in the first step. The HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% and when Memory is used more than 500Mi. The hpa file can be found as "jenkins-hpa.yaml"

$ kubectl create -f jenkins/jenkins-hpa.yaml --namespace=jenkins

Step 10: To get HPA info and description and check that resource metrics data:

$ kubectl get hpa

NAME          REFERENCE                TARGETS                     MINPODS   MAXPODS   REPLICAS    AGE
jenkins-hpa   Deployment/jenkins  1253376 / 500Mi, 0% / 50%          1         10        3          6m


$ kubectl describe hpa

Name:                                                  jenkins-hpa
Namespace:                                             jenkins
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 18 Jun 2019 08:21:16 +0200
Reference:                                             Deployment/jenkins
Metrics:                                               ( current / target )
  resource memory on pods:                             1253376 / 500Mi
  resource cpu on pods  (as a percentage of request):  0% (0) / 50%
Min replicas:                                          1
Max replicas:                                          10
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    the last scale time was sufficiently old as to warrant a new scale
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from memory resource
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:           <none>





##Create a namespace and deploy the mediawiki application (or any other application which you think is more suitable to showcase your ability, kindly justify why you have chosen a different application) on the cluster.

Step 1: Create a namespace for the mediawiki application

$ kubectl create ns mediawiki

Step 2: Now create a deployment and service defination file which contains both the mysql and mediawiki configurations. It can be found : "mediawiki/mediawiki.yaml"

Step 3: Apply the mediawiki yaml files

$ kubectl apply -f mediawiki/mediawiki.yaml

Step 4: Verify that the service is up and running

$ kubectl get services --namespace mediawiki

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
mediawiki-svc   NodePort    10.xxx.xxx.111   <none>        80:31227/TCP   57m
mysql-svc       ClusterIP   10.xxx.xx.190    <none>        3306/TCP       57m

Step 5: Access the service from the URL: http://<ip>:<port>/index.php/Main_Page





##Setup a private docker registry to store the docker images. Configure restricted access between cluster to registry and Cluster to pipeline.

Step 1: As we are using AWS so if you want the registry to be persistent, this will require a persistent volume of some kind. I’ll use the example of Elastic Block Storage to provide persistent storage:

$ aws ec2 create-volume --availability-zone $AZ --size $VOL_SIZE --volume-type gp2

Note the volumeID that is generated. We will be using that in  the resulting block in our Kubernetes YAML

Step 2: Since the Docker Registry is, itself, deployed as a Docker container, setting up the rest of your YAML file is pretty straightforward. I have put up a "registry.yaml" file that contains the necessary configurations.

Step 3: As Jenkins is running in the "jenkins" namespace so we can deploy the registry deployment and service in the same namespace.

$ kubectl apply -f dockerRegistry/registry.yaml


Step 4: Once we apply the same we would get the following endpoint.

resulting endpoint: registry.jenkins.svc.cluster.local

Step 5: I only intended the registry to be exposed to resources in-cluster (for example, if the portion of your pipeline that builds containers is within the cluster, an ingress isn’t required to make the push) So lets create a ingress controller with tls enabled. The configurations cane be found in the file dockerRegistry/tls.yaml. This will restrict the access only to the cluster and the registry.

Step 6: Lets apply the tls configs.

$ kubectl apply -f dockerRegistry/tls.yaml

Step 7: If you want to create the user, password and other details we cn use the below commnad and apply.

$ kubectl create secret docker-registry --dry-run=true $secret_name \
--docker-server=registry.jenkins.svc.cluster.local \
--docker-username=user \
--docker-password=password \
--docker-email=snigdha.sambit.ak@gmail.com -o yaml > docker-secret.yaml

$ kubectl apply -f docker-secret.yaml --namespace jenkins






##Deploy an open source vulnerability scanner for docker images scanning within the CI build pipeline.


We can use a Helm chart that bootstraps a vulnerability scanner deployment on a Kubernetes cluster using the Helm package manager. We shall be using the Anchore Engine here.

Step 1: Configure helm in your local system

$ brew install kubernetes-helm


Step 2: Configure Helm access with RBAC

Helm relies on a service called tiller that requires special permission on the kubernetes cluster, so we need to build a Service Account for tiller to use. We’ll then apply this to the cluster.

To create a new service account manifest:

cat <<EoF > rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF

This file can be found under "helm/rbac.yaml"

Next apply the config:

$ kubectl apply -f rbac.yaml


Step 3: Now we can install tiller using the helm tooling

$ helm init --service-account tiller

This will install tiller into the cluster which gives it access to manage resources in your cluster.

To update Helm’s local list of Charts, run:

helm repo update


Step 4: We can now install Anchore Engine in Kubernetes with Helm using the Anchore Engine Helm chart. The vaues file can be found: "anchoreEngine/anchorevalues.yaml". To do so, execute:

$ helm install --name anchore-stack anchoreEngine/anchorvalues.yaml --namespace jenkins

Once deployed, it will print a list of instructions and useful commands to connect to the service.


Step 5: Now we can check that all the pods are up and running:

$ kubectl get pods --namespace jenkins

NAME                                                  READY     STATUS    RESTARTS   AGE
anchore-stack-anchore-engine-core-5bf44cb6cd-zxx2k    1/1       Running   0          38m
anchore-stack-anchore-engine-worker-5f865c7bf-r72vs   1/1       Running   0          38m
anchore-stack-postgresql-76c87599dc-bbnxn             1/1       Running   0          38m



Step 6: Then, follow the instructions printed to screen to spawn an ephemeral container that has the anchore-cli tool:

$ ANCHORE_CLI_USER=admin

$ ANCHORE_CLI_PASS=$(kubectl get secret --namespace default anchore-stack-anchore-engine -o jsonpath="{.data.adminPassword}" | base64 --decode; echo)

$ kubectl run -i --tty anchore-cli --restart=Always --image anchore/engine-cli --env ANCHORE_CLI_USER=admin --env ANCHORE_CLI_PASS=${ANCHORE_CLI_PASS} --env ANCHORE_CLI_URL=http://anchore-stack-anchore-engine.jenkins.svc.cluster.local:8228/v1/
/ anchore-cli system status


Step 7: To configure Anchore to scan your private Docker repositories, use the registry subcommand for all the registry-related operations.

$ anchore-cli registry add registry.jenkins.svc.cluster.local <user> <password>

$ anchore-cli registry list

Registry                                   Type             User
registry.jenkins.svc.cluster.local        docker_v2         admin


Step 8: Once that’s complete, we can start pulling and scanning images from the private register:

$ anchore-cli image add registry.jenkins.svc.cluster.local/user/secureclient:latest

Image Digest: sha256:dda434d0e19db72c3277944f92e203fe14f407937ed9f3f9534ec0579ce9cdac
Analysis Status: analyzed
Image Type: docker
Image ID: 5e1be4e7763143e8ff153887d2ae510fe1fee59c9a55392d65f4da73c9626d76
Dockerfile Mode: Guessed
Distro: ubuntu
Distro Version: 18.04
Size: 127739629
Architecture: amd64
Layer Count: 6

Full Tag: registry.jenkins.svc.cluster.local/user/secureclient:latest



Step 9: We can also do the same step from within jenkins:

We can use the “Anchore Container Image Scanner Plugin” available in the official plugin list that we can access via the Jenkins interface.

Once we have installed the plugin and configured the connection with the Engine, we can include the Anchore evaluation as another (mandatory) step of our pipeline.

This way, the catalog will be automatically up to date and you will be able to detect and retract insecure images before they reach your Docker registry and automated functional tests are run.

Step 10: From the Manage Jenkins -> Configure System menu, you need to configure the connection with the Engine API endpoint and credentials:

In Engine mode:

Engine URl : http://anchore-stack-anchore-engine.jenkins.svc.cluster.local:8228/v1
Engine Username : admin
Engine Password: password


And as the last step of our build pipeline, we can write the image name, tags and (optionally) Dockerfile path to a workspace local file “anchore_images”:

In a shell we can write:

$ echo "$IMAGENAME:$TAG ${WORKSPACE}/Dockerfile " > anchore_images

After that, you can invoke the Anchore container image scanner in the next build step:

Anchore build options:

image list file : anchore_images

Leave the rest of the options with the default settings.

The build will fail if Anchore detects any stop build vulnerabilities. Of course, you also can define what triggers a stop.
