# devopsassignment

## CREATE A KUBERNETES CLUSTER ON GCP (GCP GIVES FREE CREDITS ON SIGNUP SO THOSE SHOULD SUFFICE FOR THIS EXERCISE) ON THEIR VIRTUAL MACHINES (DO NOT USE GKE) OR USE THE CLUSTER CREATED IN THE LEVEL2 TEST. IF POSSIBLE SHARE A SCRIPT / CODE WHICH CAN BE USED TO CREATE THE CLUSTER.

I am using Terraform for creating a kubernetes cluster on AWS Platform.

Step 1: Create the modules for the creation of the eks cluster(POP-DEV), master nodes, VPC and providers. Everything can be found in the root directory. the modules path is : "modules/eks/"

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

$ terraform plan -destroy -out POP-DEV-destroy-tf

$ terraform apply "POP-DEV-destroy-tf"





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

NAME                       READY     STATUS    RESTARTS    AGE
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

$ kubectl get hpa --namespace=jenkins


NAME          REFERENCE                TARGETS                     MINPODS   MAXPODS   REPLICAS    AGE
jenkins-hpa   Deployment/jenkins  1253376 / 500Mi, 0% / 50%          1         10        3          6m


$ kubectl describe hpa --namespace=jenkins

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

$ kubectl apply -f mediawiki/mediawiki.yaml --namespace mediawiki

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

To create a new service account manifest. This file can be found under "helm/rbac.yaml"

Next apply the config:

$ kubectl apply -f rbac.yaml


Step 3: Now we can install tiller using the helm tooling

$ helm init --service-account tiller

This will install tiller into the cluster which gives it access to manage resources in your cluster.

To update Helm’s local list of Charts, run:

$ helm repo update

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



##Setup Nginx Ingress Controller manually. Configure proper routes between ingress and the application.


Step 1: All resources for Nginx Ingress controller will be in a separate namespace, so let's create it:

$ kubectl create namespace ingress


Step 2: The second step is to create a default backend endpoint. Default endpoint redirects all requests which are not defined by Ingress rules: "nginxIngress/backend.yaml". It contaits the deployment and service for the backend.

$ kubectl create -f backend.yaml -n=ingress


Step 3: Now we need to create secrets to specify the SSL certificate for Nginx

We can create a self-signed certificate using OpenSSL. The common name specified while generating the SSL certificate should be used as the host in your ingress config. We create secrets for the given key, certificate and dhparam files. Use corresponding file names for the key, certificate and dhparam.

$ kubectl create secret tls tls-certificate --key <key-file>.key --cert <certificate-file>.crt -n=ingress
$ kubectl create secret generic tls-dhparam --from-file=<dhparam-file>.pem -n=ingress

Step 3: Now we can create a service for the controller. The service is of type LoadBalancer so that it is exposed outside the cluster.

The config consists of the deployment and service for the nginx-ingress.

The secret for the default SSL certificate and backend service are passed as args.
The image nginx-ingress-controller:0.9.0-beta.5 is used.
The tls-dhparam secret is mounted on a volume to be used by the controller.


Create the controller by running:

$ kubectl create -f nginx-controller.yaml -n=ingress


Step 4: Now lets create 2 sample apps that we can use for testing the ingress controller.

I have created 2 app definations which can be found : "nginxIngress/app1.yaml" and "nginxIngree/app2.yaml"

Lets create these Resources:

$ kubectl create -f app1.yaml -n=ingress

$ kubectl create -f app2.yaml -n=ingress


Step 5: Now lets define Ingress rules for load balancer status page: This can be found in the file nginxIngress/nginx-ingress.yaml

$ kubectl create -f nginxingress.yaml -n=ingress

Step 6: Now lets create Ingress rules for the sample web apps.
This file can be found : nginxIngress/app-ingress.yaml

$ kubectl create -f app-ingress.yaml -n=ingress

Notice the nginx.ingress.kubernetes.io/rewrite-target: / annotation. We are using /app1 and /app2 paths, but the apps don’t exist there. This annotation redirects requests to the /.

Step 7: Add test.dev.com domain to hosts file:
$ echo "127.0.0.1 test.akomljen.com" | sudo tee -a /etc/hosts

Step 8: You can verify everything by accessing at those endpoints:

http://test.dev.com:30000/app1
http://test.dev.com:30000/app2
http://test.dev.com:32000/nginx_status



## Setup Istio and configure Kiali &amp; Zipkin.

Step 1: Before we can get started configuring Istio we’ll need to first install the command line tools that you will interact with.

$ curl -L https://git.io/getLatestIstio | sh -

// version can be different as istio gets upgraded
$ cd istio-*

$ sudo mv -v bin/istioctl /usr/local/bin/

Step 2: We need to install Helm and Tiller before installing istio

First create a service account for Tiller.

kubectl apply -f install/kubernetes/helm/helm-service-account.yaml

Step 3: Next we need to install the Custom Resource Definitions, also known as CRDs are API resources which allow you to define custom resources.

$ helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system

We can check the installation by running:

$ kubectl get crds --namespace istio-system | grep 'istio.io'

This will return the crds

Step 4: Now lets install Istio’s core components along with Kiali and Zipkin.

This can be done in 2 ways:

1. We can modify the values.yaml file which is present in the location "install/kubernetes/helm/istio/values.yaml" and modify the following values and set them to true :

grafana:
  enabled: true

prometheus:
  enabled: true

tracing:
  enabled: true

kiali:
  enabled: true

This will enable grafana, prometheus, tracing and Kiali in our istio system.

Then we can use helm to install istio as follows:

$ helm install install/kubernetes/helm/istio --name istio --namespace istio-system

2. We can also enable zipkin, kiali, grafana and prometheus directly from the commandline.

$ helm template --set kiali.enabled=true --set tracing.enabled=true --set tracing.ingress.enabled=true --set tracing.provider=zipkin --set grafana.enabled=true --set prometheus.enabled=true install/kubernetes/helm/istio --name istio --namespace istio-system > $HOME/stash/devopsassignment/istio-1.2.0/istio.yaml

Now we will get the "istio.yaml" file inside the istio-1.2.0 folder.

We can apply the same file:

$ kubectl apply -f $HOME/stash/devopsassignment/istio-1.2.0/istio.yaml

Step 5: You can verify that the services have been deployed and check the corresponding pods

$ kubectl get svc -n istio-system

$ kubectl get pods -n istio-system

NAME                                    READY     STATUS      RESTARTS   AGE
grafana-7b46bf6b7c-4rh5z                1/1       Running     0          10m
istio-citadel-75fdb679db-jnn4z          1/1       Running     0          10m
istio-galley-c864b5c86-sq952            1/1       Running     0          10m
istio-ingressgateway-668676fbdb-p5c8c   1/1       Running     0          10m
istio-init-crd-10-zgzn9                 0/1       Completed   0          12m
istio-init-crd-11-9v626                 0/1       Completed   0          12m
istio-pilot-f4c98cfbf-v8bss             2/2       Running     0          10m
istio-policy-6cbbd844dd-ccnph           2/2       Running     1          10m
istio-telemetry-ccc4df498-pjht7         2/2       Running     1          10m
prometheus-89bc5668c-f866j              1/1       Running     0          10m
kiali-5d4b49848-qvdtr                   1/1       Running     0          10m
zipkin-6cbbd844dd-khzn9                 1/1       Running     0          10m


Step 4: For testing (and temporary access), you may also use port-forwarding. Use the following, assuming you’ve deployed Zipkin to the istio-system namespace:

$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=zipkin -o jsonpath='{.items[0].metadata.name}') 15032:9411

Now in the local system zipkin dashboard is available in http://localhost:15032.

Step 5: For testing (and temporary access), you may also use port-forwarding. Use the following, assuming you’ve deployed Kiali to the istio-system namespace:

$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001

Now in the local system Kiali UI is available in http://localhost:20001/kiali/console.

To log into the Kiali UI, go to the Kiali login screen and enter the username and passphrase which is by default asmin:admin


## Setup mTLS authentication between microservices. Use self-signed certificates for secure communication between microservices.


Istio can secure the communication between microservices without requiring application code changes. Security is provided by authenticating and encrypting communication paths within the cluster.

Istio Citadel is an optional part of Istio's control plane components. When enabled, it provides each Envoy sidecar proxy with a strong (cryptographic) identity, in the form of a certificate. Identity is based on the microservice's service account and is independent of its specific network location, such as cluster or current IP address. Envoys then use the certificates to identify each other and establish an authenticated and encrypted communication channel between them.

Citadel is responsible for, Providing each service with an identity representing its role, Providing a common trust root to allow Envoys to validate and authenticate each other and Providing a key management system, automating generation, distribution, and rotation of certificates and keys.

When an application microservice connects to another microservice, the communication is redirected through the client side and server side Envoys. The end-to-end communication path is:

1. Local TCP connection between the application and Envoy (client- and server-side);

2. Mutually authenticated and encrypted connection between Envoy proxies.

When Envoy proxies establish a connection, they exchange and validate certificates to confirm that each is indeed connected to a valid and expected peer.

Step 1 : For Istio to work, Envoy proxies must be deployed as sidecars to each pod of the deployment. There are two ways of injecting the Istio sidecar into a pod: manually using the istioctl CLI tool or automatically using the Istio sidecar injector. Here we will use the automatic sidecar injection provided by Istio.


$ kubectl label namespace default istio-injection=enabled

$ kubectl get namespace -L istio-injection

NAME             STATUS   AGE    ISTIO-INJECTION
default          Active   2h      enabled
istio-system     Active   3h
...

Step 2: Create 2 sample apps called app1.yaml and app2.yaml which can be found under the mTLS folder and deploy those in the default namespace.

$ kubectl apply -f mTLS/app1.yaml

$ kubectl apply -f mTLS/app2.yaml

Step 3: Verify setup by sending an http request (using curl command) from any app1 pod to app2. All requests should success with HTTP code 200

$ for from in "default"; do kubectl exec $(kubectl get pod -l app=app1 -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl http://app1.default:5000/ip -s -o /dev/null -w "app2.${from} to app1.default: %{http_code}\n"; done

app2.default to app1.default: 200


Step 4: Citadel is Istio's in-cluster Certificate Authority and is required for generating and managing cryptographic identities in the cluster. Verify Citadel is running:

$ kubectl get deployment -l istio=citadel -n istio-system

Expected output:

NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-citadel   1         1         1            1           15h


Step 5: Define mTLS authentication policy for the app1 service. This file can be found as "mTLS/app1-authentification.yaml"

$ kubectl create -f mTLS/app1-authentification.yaml

policy.authentication.istio.io/mtls-to-app1 created

Confirm the policy has been created:

$ kubectl get policies.authentication.istio.io

NAME              AGE
mtls-to-app1      1m


Step 6: Now we need to enable mTLS from app2 using a Destination rule. This file can be found as "mTLS/destination-rule.yaml"

$ kubectl create -f destination-rule.yaml

Output:
```
destinationrule.networking.istio.io/route-with-mtls-for-app1 created
```

Step 5: If mTLS is working correctly, the app2 should continue to operate as expected, without any user visible impact. Istio will automatically add (and manage) the required certificates and private keys





## Setup Kubernetes Dashboard and secure access to the dashboard using a read only token

Step 1: Deploy the Kubernetes dashboard to your cluster. I have it stores in the path "kubernetesDashboard/kubernetes-dashboard.yaml"

$ kubectl apply -f kubernetesDashboard/kubernetes-dashboard.yaml

secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created

Step 2: Since this is deployed to our private cluster, we need to access it via a proxy. Kube-proxy is available to proxy our requests to the dashboard service.

$ kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &

This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests. This command will continue to run in the background of the current terminal’s session.

Step 3: Now we can access the Kubernetes Dashboard using the url : http://localhost:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

Step 4: Now to get the read only token we need to open a New Terminal Tab and use the aws-iam-authenticator with the clusted id:

$ aws-iam-authenticator token -i POP-DEV

{"kind":"ExecCredential","apiVersion":"client.authentication.k8s.io/v1alpha1","spec":{},"status":{"token":"k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSahjvahsdvFo1UkRZQkRBJTJGMjAxOTA2MTklMkZ1cy1lYXN0LTElMkZzdHMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDE5MDYxOVQxNjM1MjlaJlgtQW16LUV4cGlyZXM9NjAmWC1BbXotU2lnbmVkSGVhhhb3N0JTNCeC1rOHMtYXdzLWlkJlgtQW16LVNpZ25hdHVyZT1kMWI2NGU2MzMwMDY3YTFkY2I3ZjMyZDkzZmFmZDRhYmU1MGRkZTA2YWNhYjgwNjRiYzE3YjVkMTlmYTU3ZDE0"}}

Step 5: Copy the output of this command and then click the radio button next to Token then in the text field below paste the output from the last command.
