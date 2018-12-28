# Udemy Course - Learn Devops: Advanced Kubernetes Usage

* [Course](https://www.udemy.com/learn-devops-advanced-kubernetes-usage/)
* [Repository](https://github.com/wardviaene/advanced-kubernetes-course)

## Section 1 - Introduction

### Lecture 1 - Introduction

* this course is for those you have done the LearnDevops: The Complete Kubernetes Course
* Topics Covered:
	* Logging: ElasticSearch (as StatefulSet), fluentd, Kibana, LogTail
	* Authentication: OpenID Connect (OIDC), Auth0
	* Packaging: Helm
	* The Job Resource: Job
	* Scheduling: CronJob
	* Deploying on Kubernetes: Spinnaker
	* Microservices on Kubernetes: Linkerd
	* Federation: kubefed
	* Monitoring: Prometheus
* Course objectives:
	* be able to use advanced k8s features
	* use K8s in an enterprise env
	* apply security feats in k8s
	* setup a high availability cluster (using federation)

## Section 2 - Centralized Logging

### Lecture 4 - Logging in Kubernetes

* logging is important to show errors, info and debugging data about the app
* when we run one app it is easy to look for the logs. we open app.log file. if the app is deployed in a pod we use `kubectl logs <pod>`
* In k8s 1 app will run as one or many pods. in that case debugging is more difficult. we dont know in which pod to look in
* with what we know so far we have to look up: pod name. container names, run `kubectl pod` for every container
* to solve this problem we can do Log Aggregation
* this approach is used for years (e.g syslog since 80s)
* Log aggregation nowdays is done with modern tools such as ELK stack (ElasticSearch, Logstash, Kibana) (selfhosted)
* Hosted services (loggly.com, papertrailapp.com)
* To do centralized logging we will use:
	* Fluentd (log forwarding)
	* ElasticSearch (log indexing)
	* Kibana (visualization)
	* Logtrail (UI to show logs, a plugin for kibana)
* this stack can be customized to create custom dashboards

### Lecture 5 - Logging with Fluentd + ElasticSearch + Kibana + LogTrail (part I)

* we goto CourseRepo/logging where we have README.md with all the setup commands.
* we will use kops to spin a cluster on AWS through vagrant
* we launch vagrant 
```
vagrant uo
vagrant ssh
``` 
* we clone the course -repo in vagrant `git clone https://github.com/wardviaene/advanced-kubernetes-course.git`
* we got to `cd advanced-kubernetes-course/logging`
* we spin up a cluster with medium sized nodes (3). its pretty expensive so we have to finish fast.
* we use the basic course s3 state repo and domain in rout53 `kops create cluster --name=k8s.agileng.io --state=s3://kops-state-4213432 --zones=eu-central-1a --node-count=3 --node-size=t2.medium --master-size=t2.micro --dns-zone=k8s.agileng.io`
* we update `kops update cluster k8s.agileng.io --yes --state=s3://kops-state-4213432`
* first script we look at is es-statefulset.yaml. it sets a ServiceRole, ClusterRole and CLusterRoleBinding and a StatefulSet with elasticsearch (2 replicas) with resource limits and storage (volume) using standard storage defined in storage.yml (8gb) for each replica. make susre to set correct availability zone to be same with cluster we use StaticSet instead of Deployment when we need static hostnames. ElasticSearch needs the static hostnames to work. th pods of ES launch an initcontainer to run a priviledge command to set kernel param
* ES has another config WAMl es-service.yaml to set a Service for ES exposing port 9200
* we have fluentd-es-ds.yaml for fluentd deployment. we have authorization with RBAC. and a DeamonSet deployment. to run a signle pod on each node. a lot of env aparams and volumes
* the template for fluentd has a nodeselector
* we have a fluentd-config-map.yaml. with config of how logs are mapped and config of fluentd. it captures input and converts it to JSON format to feed it to elasticsearch
* to view the logs we use kibana kibana-deployment.yaml. it is a deployment of tutors custom image (kibana+logtrail)
* XPACK monitoruing and security is disabled (in free version) we need to implement our own authentication on loadbalancer or filter ips as loadbalancer is public

### Lecture 6 - Logging with Fluentd + ElasticSearch + Kibana + LogTrail (part II)

* `kops version` is 1.10.0
* `kubectl get nodes` gives me the 3 nodes
* i label the nodes `for i in `kubectl get node |cut -d ' ' -f 1 |grep internal` ; do kubectl label nodes ${i} beta.kubernetes.io/fluentd-ds-ready=true ; done`
* i execute all scripts `kubectl create -f .`
* all pods are in kube-system namespace `kubectl get pods --namespace=kube-system`
* we go to AWS to see the instances. we have 1 LB for kibana. we can add it to ROUTe53
* we need to edit security groupo and add our IP address. we add ELB security group edit inbound rules limiting to only our ip. we visit `http://a1bf4c60b07d011e984300210e252538-1878325862.eu-central-1.elb.amazonaws.com:5601`
* we see kibana landpage. we set indexname to logstash-* as fluentd is a replacement for logstash and click create
* we select discover. we see the jsons of the logs created
* we click logtrail plugin and see a combined log trail. we can filter and use kibana to customize it

## Section 3 - Authentication

### Lecture 7 - HTTP Basic Authentication in Kubernetes

* by default X509 certificates are used to authenticate yourself to the kubernetes api-server (with kubectl)
* these certs were issued when we created the cluster for the first time
* if we use minikube or kops this is done for us (first thing kops does)
* in first course we saw how to create new users generating new certs and siging them with the Cert Authority used by k8s
* k8s api server is based on http. one option is to use http basic auth. HTTP Basic Auth only requires us to send a username and password to the API server
* While simple for the user. a user-password combo is less secure and difficult to maintain in k8s
* to enable basic auth we can use a static password file on the K8s master
* the path to this static password file needs to be passed to the apiserver as an argument `--basic-auth-file=/path/to/somefile`
* the file needs to be formatted as `password,user,uid,"group1,group2,group3"`
* the basic auth has downsides:
	* its only supported for convenience. k8s team works on making teh more secure methods easier to use
	* to add (or edit) a user the api-server needs to be restarted

### Lecture 8 - Authentication using a Proxy

* another way to handle authentication is to use a proxy
* when using a proxy we can handle the authentication part ourselves
* we can write our own authentication mech and provide the username, and groups to the k8s API once the user is authenticate
* this is agood solution if the k8s does not support the authentication method we want
* Proxy setup needs the following steps:
	* proxy needs a client cerificate signed by the cert authority that is passed to the api server using --requestheader-client-ca-file
	* the proxy needs to handle the authentication (using a form, basic auth, or another mech)
	* once the user is authenticated the proxy needs to forward the request to the k8s API server and set a HTTP header with the login
	* this login http header is determined by a flag passed to the API server e.g: --requestheader-username-headers=X-Remote-User. in this case the proxy needs to set the X-Remote-User after authentication
	* --requestheader-group-headers=X-Remote-Group can be used as an argument to set the group header
	* --requestheader-extra-headers-prefix=X-Remote-Extra- allows us to set extra headers with extra info about the user

### Lecture 9 - Authentication using OpenID Connect (IODC)

* another (better) alternative is to use OpenID Connect tokens
* OpenID Connect is built on top of OAuth2
* it allows us to securely authenticate and then receive an ID Token
* this ID token can be verified whether it really originated from the authentication server, because its signed (using HMAC SHA256 or RSA)
* This ID token is a JWT. it contains known fields like username and optinally groups
* Once this token is obtained it can be used as credential to authenticate to the apiserver
* We can pass --token=<ourtoken> when executing kubectl commands
* kubectl can automatically renew our token_id when it expires (this does not work with all identity providers)
* with this token we can also authenticate to the K8s UI (to make it easier in demos a reverse proxy is created that can authenticate us with OpenID Connect and then pass the token to the UI)
* The flow is: 
	* User=>Identity Provider: Login to IdP
	* IdP=>User: provide access_token, id_token and refresh_token
	* User=>Kubectl: call kubectl with token being the id_token OR add tokens to. kubectl/config
	* kubectl=> API server: Authorization: Bearer...
	* APi server: is JWT signature valid?
	* API server: Has the JWT expired? (iat+exp)
	* API server: User Authorized?
	* API server=> kubectl: Authorized: perform action and return result
	* Kubectl=>User: Return result

### Lecture 10 - Demo: OIDC with Auth0 (Part I)

* Steps to take
	* setup identity Provider (auth0 account)
	* create auth0 client for kubernetes
	* setup cluster withoidc (OpenID connect) using kops edit
	* deploy authentication server (for UI proxy + to hand out bearer tokens)
	* change env variables in deployment to match auth0
	* create DNS record for auth server
	* try log in to k8s UI through authentication server
* we go to [auth0](https://auth0.com) a sit has a free tier. 
	* we signup
	* we create a new application => reegular web app (name=Kubernetes) => settings
	* we get a domain, client id and client secret. the client secret needs to go to our auth server
	* we have to specify the allowed callback urls (http://authserver.k8s.agileng.io/callback)
	* we have to specify the allowed logout urls (http://authserver.k8s.agileng.io)
	* we dont care about http in our loadbalancer we can redirect to https
	* save changes
* we need the domain url from auth0 in our cluster to spin in AWS
* i ssh in vagrant vm and go to /advanced-kubernetes-course/authentication
* it has a REAADME.md with instructions how to create a k8s ui and how to edit the cluster adding oidc
* we c4reate the cluster `kops create cluster --name=k8s.agileng.io --state=s3://kops-state-4213432 --zones=eu-central-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=k8s.agileng.io`
* we edit the cluster `kops edit cluster k8s.agileng.io --state=s3://kops-state-4213432` and add 
```
spec:
  kubeAPIServer:
    oidcIssuerURL: https://achliopa.eu.auth0.com/
    oidcClientID: clientid #use your id from auth0
    oidcUsernameClaim: sub
```
* we update the cluster to save changes

### Lecture 11 - Demo: OIDC with Auth0 (Part II)

* we wait till cluster is updated and create the K8s UI using `kubectl create -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/kubernetes-dashboard/v1.6.3.yaml`
* we have a Secrets YAML definition'auth0-secrets.yaml' where we add the client secretof auth0
* we have the auth0-service.yaml which is a Loadbalancer listening to port 80
* we have the auth0-deployment.yaml which is the Deployment of the k8s auth server (1 replica of tutors custom image using a number of env params which need to be adapted to our cluster)
* we need to base64 encode the client secret before adding it to secrets `echo -n "<SECRET>" |base64`
* what we are missiing is the  AUTH0_CONNECTION in auth0-deployment.yaml. in auth0.com in the application settings we go to  Connections => database. the name we put in should be the same as the value in the YAML file 'Username-Password-Authentication'
* we need to eanable it. we to to applications => Kubernetes => Connections +> database => enable
* we can create users in auth0. we create 1 for testing
* we apply all in the repo folder. we look for pods. when it starts we check for services. we see the LB. we go to ROute53 and add it in hosted zone authserver.k8s.agileng.io (we can add tls cer with cert manager in aws)
* the code for kubernetes-auth-server (python with flask) is in /advanced-kubernetes-course/kubernetes-auth-server. in auth0.com => clients there a re sample code for various tech (nodejs, go)
* we hit authserver.k8s.agileng.io and we see the ui. pass in the login and get redirected to auth0. after succesfully loegd in we go to authserver.k8s.agileng.io/dashboard
* we see our token content
* if we click access ui. the auth server will reverse proxy to k8s api setting the token a shttp header
* with the token k8s api server will validate our access

### Lecture 12 - Demo: OIDC with Auth0 (Part III)

* we will use kubectl to test auth0 on kubernetes.
* we go to advanced-kubernetes-course/kubernetes-auth-server
* we need to have python. we check `python3 --version` and `python --version`
* we check dependencies `cat reuirements-cli.txt`
* we install pip3 `sudo apt-get install python3-pip`
* we install dependencies `pip3 install -r requirements-cli.txt`
* solving locale issue with python `export LC_ALL=C`
* we execute cli `./cli-auth.py` . we need to pass in as env vars AUTH0_CLIENT_ID AUTH)_DOMAIN APP_HOST `AUTH0_CLIENT_ID=Px5toCAsq3AzkdY2Q2M9m44ywhKIRjZe AUTH0_DOMAIN=achliopa.eu.auth0.com APP_HOST=authserver.k8s.agileng.io ./cli-auth.py` i login and get an error
* in auth0 app settings we go to advanced settings and enable grant types-> password
* we retry and we get an id token
* this token is saven in .kube/id_token and in .kube/jwks.json
* we can use the token with kubectl. to not write this token everytime we make an alias for kubectl `alias kubectl="kubectl --token=\$(AUTH0_CLIENT_ID=Px5toCAsq3AzkdY2Q2M9m44ywhKIRjZe AUTH0_DOMAIN=achliopa.eu.auth0.com APP_HOST=authserver.k8s.agileng.io ~/advanced-kubernetes-course/kubernetes-auth-server/cli-auth.py)"`
* first we remove all users with `vim ~/.kube/config` (everything under users:)
* we write `kubectl get nodes` login and we see the nodes.
* all these are explaned in k8s documentation (authenticaiton -> openid tconnect token) (option 2)
* if we dodn't use the convenience script wwe would have to insert the token explicitly

## Section 4 - Authorization

### Lecture 13 - Introduction to Authorization

* authorization controls what the user can do. where does the user has access to
* access control is implemented on API level (kube-apiserver)
* when an API req comes in. (e.g when we enter kubectl get nodes) it will check whether we have access to execute the command
* multiple authorization modules available:
	* Node: a special purpose auth mode that authorizes API reqs made by kubelets
	* ABAC: Attribute Access COntrol (access rights controlled by policies that combine attributes)
	* RBAC: Role base access control: regulates access using roles. can dynamixcally config permission policies
	* Webhook: sends authorization req to external REST interface (e.g if we have our own auth server). parse incoming JSON and rep with access granted or denied
* To enable authorization mode we need to pass --authorization-mode= to the API at startup
* when using kops we can create a cluster with flag --authrization. if we dont spec the flag. we allow everything. like specing --authorization AlwaysAllow
* when specing --authorization RBAC the cluster will use RBAC
* in running clusters we can edit (kops edit) and add in spec:
```
kubeAPIServer:
	authorizationMode: RBAC
```
* in minikube at start `minikube start --extra-config=apiserver.Authorization.Mode=RBAC`

### Lecture 14 - Introduction to RBAC

* we can add RBAC resources with kubectl to grant permnissions (YAML => apply) 
	* we define a Role, and assign users/groups to the role
	* we can crerate roles for a namespace or crossnamespace, cluster roles
* Resources for RBAC:
	* Role (single namespace) or CLusterRole(cluster-wide)
	* RoleBinding or CLusterRoleBinding to assign usersgroups to the role
* A Role has a namespace: a ROle or ClusterRole have name: and rules:
* in RoleBindings we have subjects: (kind: User or Group) and roleRef: )the role we bind to

### Lecture 15 - Demo: authorizations - RBAC

* RBAC demo with auth0
* we are in vagrant vm and we cd to advanced-kubernetes-course/authorization
* in README.md we see the instructions
* we spin a cluster `kops create cluster --name=k8s.agileng.io --state=s3://kops-state-4213432 --zones=eu-central-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=k8s.agileng.io --authorization RBAC` and edit it to add auth0
```
spec:
  kubeAPIServer:
    oidcIssuerURL: https://achliopa.eu.auth0.com/
    oidcClientID: Px5toCAsq3AzkdY2Q2M9m44ywhKIRjZe
    oidcUsernameClaim: name
    oidcGroupsClaim: http://authserver.k8s.agileng.io/claims/groups
  authorization:
    rbac: {}

```
* we update the cluster
* we need to add configuration in auth0 => Application => extensions => authorization (Auth0 Authorization) => install => set our default user email => groups => create first group (name=developers)=> add members => our default user => save
* click top right (our domain) => configuration => enable groups => publish rule
* app => rules => cretate new rule => empty rule => del all code and replace with 
```
function (user, context, callback) {
  var namespace = 'http://authserver.k8s.agileng.io/claims/'; // You can set your own namespace, but do not use an Auth0 domain

  // Add the namespaced tokens. Remove any which is not necessary for your scenario
  context.idToken[namespace + "permissions"] = user.permissions;
  context.idToken[namespace + "groups"] = user.groups;
  context.idToken[namespace + "roles"] = user.roles;
  
  callback(null, user, context);
}
```
* this adds the group to the oidc token => save
* we need to deploy our auth server from advanced-kubernetes-course/authentication/ `kubectl create -f .`
* we also alias elb to route53 to nameset
* we can deploy the authserver locally if we want to
* we visit 'authserver.k8s.agileng.io' and login. we see that we are in gorup developers
* we will create a role ink8s cluster and add group developers in the role. the YAML is /advanced-kubernetes-course/authorization/role.yml with a Role and RoleBinding
* we will add a second account in the vagrant vm  so that we login with cert in one case and with auth0 on other case `sudo adduser k8s-test ` we swap user `sudo su - k8s-test`
* i need to cp the ~/.kube/config from vagrant user as we want the CA of the cluster. we create a .kube/config file and cp the content up to users:
* the we have to follow the part of authorization demo to attach the token to the kubectl command
* we have forbidden msgs. we apply the rules and get access (as normal user)

### Lecture 16 - Predefined and more complex roles

* we can se more complex RBAC Role. setting multiple apiGroups each one with its own resources and verbs. this fine grains upon k8s API groups like batch, extensions, apps etc. the groups have to do with the resources they are applied on
* we can use one of the predefined roles
* Pre-Defined RBAC Roles:
	* cluster-admin: super-user access. Can be used with ClusterRoleBinding (superuser access on the cluster) or with RoleBinding to limit access within a namespace
	* admin: admin access, intended to be used only with RoleBinding. has read/write access within the namespace. but cannot change quotas
	* edit: read/write access to most objects within the namspace. but cannot view/create new roles or rolebindings
	* view:  read access/ but can't see any secrets roles or rolebindings

## Section 5 - Package Management

### Lecture 17 - Introduction to Helm

* Helm is the package manager for Kubernetes
* it helps manage Kubernetes apps
* Helm was started by Google and Deis (Deis provides a PaaS 'Platform as a Service' on K8s)
* Deis is inspired by heroku but runs on K8s
* the packaging format is called charts (a collection of files that describe a set of K8s resources)
overriding values in template YAML files in charts is useful to make sure the app is configured in a way we want

### Lecture 18 - Demo: Installing MySQL on K8s with Helm

* we spin a k8s cluster on AW S using kops fr5om vagrant vm
* we need the helm package on our local machine (have it in vagrant from previous course)
* we go to /advanced-kubernetes-course/helm in README.md we have instructions
* we run `helm init` it adds tiller in cluster `kubectl get pods -n kube-system` proves that
* we run helm install command `helm install --name my-mysql --set mysqlRootPassword=secretpassword,mysqlUser=my-user,mysqlPassword=my-password,mysqlDatabase=my-database stable/mysql` with --set we set parmas.
* we get an error `Error: release my-mysql failed: namespaces "default" is forbidden: User "system:serviceaccount:kube-system:default" cannot get namespaces in the namespace "default"` its a bug of RBAC
* we delete tiller deployment `kubectl delete deploy tiller-deploy  -n kube-system`
* we create a YAML file rbac.yml and add the code
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: ""
```
* we create - delete -create it
* we run `helm init --upgrade`
* we run install again
* NOT WORKING we will do what we did in K8s basic course as that worked

## Section 6 - The Job Resource

### Lecture 19 - Introduction to the job resource

* till now we ve seen pods as longrunning services that dont stop (like webservers)
* we can also schedule pods as Jobs rather than with ReplicationCOntroller/ReplicaSet
* with RC or RS the pod will be indefiitely restarted if it stops/fails
* with the Job resource, pods are expected to run a task and then exit
* there are 3 main types of jobs:
	* Non parallel Jobs: the Job resource will monitor the job. and restart the pod if it fails or gets deleted (we can control this with restartPolicy: attr). when pod successfully completes (exit with 0) job completes
	* Parallel Jobs with fixed completion count: a Job can run multiple pods in parallel. we can spec a fixed completion count. job completes when succesfully exited pods == completion count (completions: attr)
	* Parallel Jobs with work queue: pods should coordinate with themselves (or an external service) to determine what each should work on. when any pod terminates with success. no more pods are created and the job is considered completed. the job is considered completedif a pod terminates.

### Lecture 20 - Demo: Running a Job

* we spin acluster on AWS from vagrant
* we look at job.yml file that describes a Job resource that uses a perl pod to calculate pi
* we apply it . a pod is create runs and completes

### Section 7 - Scheduling

### Lecture 21 - Scheduling with CronJob

* a Cron job can schedule Job resources based on time
	* once at a speced time
	* recurrently
	* CronJob resource is comparable with crontab in Linux/Unix systems
* CronJob schedule format is same as crontab 
	* e.g  every night at 3:20AM : `20 3 * * *` 
	* every 15 minutes: `*/15 * * * *`
* CronJob is still in alpha. it uses schedule: attr and jobTemplate: attr much like the pods

### Lecture 22 - Demo: Using alpha resource CronJob

* CronJob resource is now in beta. NO NEED to enable it in kubeAPIserver using kops edit
* we DONT HAVE go to ./advanced-kubernetes-course/jobs  and  look at README.md for instructions. we need to mod the cluster with kops edit adding 
```
spec:
  kubeAPIServer:
    runtimeConfig:
      batch/v2alpha1: "true"
```
* we look at cronjob.yml. it schedules every 1 minute to run busybox that willecho a message
* we apply it after changing first line to `apiVersion: batch/v1beta1`
* we look at cronjobs with `kubectl get cronjob`. we look in ti with `kubectl describe cronjob/hello`
* we look in pods with --show-all and see every min anew completed pod. if we log in one we see the echo msg

## Section 8 - Deploying on Kubernetes with Spinnaker

### Lecture 23 - Introduction to Spinnaker

* Spinnaker is a CD (Continuous Delivery) platform
* It can automate deployments on different cloud providers (e.g AWS EC2, GCE, Azure, Openstack and Kubernetes)
* It works only on immutable (cloud-native) apps (e.g docker images)
* Created by Netflix
* Integrates with CI tools (Jenkins,Travis) with monitoring tools and provides different  deployment strategies
* Spinnaker has 2 core sets of features:
	* Cluster Management: View and manage our cluster resources in the cloud (or in K8s)
	* Deployment Management: Create and manage CD workflows, using a delivery pipeline
* It supports various deployment strategies:
	* Blue/Green (called Red/Black): using an LB and 2 groups of pods (deploy then switchover)
	* Rolling Blue/Green (rolling Red/Black): start with small capacity (2 pods) when they are healthy add 2 more till all are updated
	* Rolling Deployment with Canary analysis: not only health checks but in depth behaviour analysis before adding pods. Netflix does this
* An example CD pipeline in SPinnaker: Start => Find image from TEST => Deploy CANARY => Waint 30 mins + Cutover manual approval => Deploy PROD (red/black) => Tear down CANARY + WAITH 2 hrs => Destroy old PROD
* it uses 2 PROD environments PROD and CANARY
* the terminology in SPinnaker is a bit different from K8s
	* Account: maps to credentials able to authenticate agains K8s, and docker registries where your images are stored
	* Instance: maps to a K8s pod
	* Server Group: maps to a Replica Set
	* Cluster: a 'Spinnaker deployment' on K8s. Spinnaker uses its own orchestration and is different from the Deployment on K8s
	* Load Balancer: maps to a K8s service

### Lecture 24 - Deploying on Kubernetes using Spinnaker (Part I)

* we'll see how to:
	* Setup spinnaker using helm on K8s cluster
	* Configure spinnaker
	* Deploy an app on K8s from SPinnaker
* the pipeline we'll follow: commit to git repo => trigger from github a docker build => docker push => trigger spinnaker pipeline from docker registry => k8s deploy => app is live
* we go to the course repo spinnaker folder advanced-kubernetes-course/spinnaker. we look in README.md for isntructions
* we spin our cluster on AWS (we run 3 small nodes, tutor runs 1 small and 2 medium)
* we need to seup helm. in /kubernetes-course/helm we apply helm-rbac.yaml
* we run `helm init --service-account tiller`
* we check spinnaker.yaml in /advanced-kubernetes-course/spinnaker
* we need to make a few changes like our dockerhub repo our credentials
* we use protforwarding on spinnaker. to expose it we need a loadbalancer
* we run `helm install --name demo -f spinnaker.yml stable/spinnaker`
* to see the new pods for spinnaker we run get pods
* once spinnaker starts we need to connect to it using prot forwarding (kubectl port-forwarding). it always runs locally cannot bind a proxy address (--bind-adress) to hit is from browser on host while running from a vm
* spinnaker how to
```
NOTES:
1. You will need to create 2 port forwarding tunnels in order to access the Spinnaker UI:
  export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward --namespace default $DECK_POD 9000

2. Visit the Spinnaker UI by opening your browser to: http://127.0.0.1:9000

To customize your Spinnaker installation. Create a shell in your Halyard pod:

  kubectl exec --namespace default -it demo-spinnaker-halyard-0 bash

For more info on using Halyard to customize your installation, visit:
  https://www.spinnaker.io/reference/halyard/

For more info on the Kubernetes integration for Spinnaker, visit:
  https://www.spinnaker.io/reference/providers/kubernetes-v2/

```
* to overcome this given  we have kubectl installed on host we cp ~/.kube/config from vagrant vm to host
* now from host system we access the AWS cluster
* spinnaker installs jenkins
* on host system we run `export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")` and `kubectl port-forward --namespace default $DECK_POD 9000`
* now we visit with browser localhost:9000 and see spinnaker running

### Lecture 25 - Deploying on Kubernetes using Spinnaker (Part II)

* we got to github [project repo](https://github.com/wardviaene/docker-demo) it has a simple node app and a Doxckerfile
* we fork the project. we need to link it to dockerhub
* we go to dockerhub and login. we link our dockerhub account with github account docker hub build
* we create a new automated build in dockerhub
	* we create anew repo spinnaker-node-demo in dockerhub. we can trigger a build with git push. or bmanually from dockerhub
* we trigger a build from hub

### Lecture 26 - Deploying on Kubernetes using Spinnaker (Part III)

* we will configure spinnaker to use the build to do a new deployment
* in loacalhost:9000 in UI => applications => actions => create new app (nodejs) and create
* we create a load balancer (k8s service)
	* stack: dev
	* target port : 3000
	* type: ClusterIP
* after creating in the cluster we run `kubectl get svc` and see the nodejs-dev LB
* we create a new Server Group
	* stack: dev
	* containers: spinnaker-node-demo (find it from docker)
	* load balancer: nodejs-dev
	* container port: 3000
	* probes => port: 3000
	* Create
* from host we run `kubectl proxy` we get a proxy for the API. we visit the URL to see if app is running
* in this way we can avoid using load balancer just for testing

### Lecture 27 - Deploying on Kubernetes using Spinnaker (Part IV)

* we will create the pipeline
	* name=> trigger-deploy-to-dev
	* automated trigger: Docker Registry
	* registry name: dockerhub
	* organization: achliopa
	* image: achliopa/spinnaker-node-demo
* add stage
	* type: Deploy
	* Stage-name: Deploy-to-dev
	* add servergoup (nodejs-dev)
	* image: tag resoved at runtime (latest)
	* ADD
* add stage
	* type: Destroy Server Group
	*  stage name: Destroy Server Group
	* cluster: nodejs dev
	* target: previous server group
* we test bu pushing a change to github repo (separate branch) and do pull request on github for master

## Section 9 - Linkerd

### Lecture 28 - Introduction to Linkerd

* on K8s we can run a lot of microservices on one cluster
* it can quickly become difficult to manage the endpoints of all the different services that make up an app within the cluster
	* service discovery in a cluster is limited
	* routing is often on a round-robin basis 
	* no failure handling (only removing pods that fail the healthcheck)
	* difficult to visualize the different services
* HTTP APIs can bee very basic and there is no failure handling (what id app A send HTTP GET to App B but is down? failed req no retry)
* Linkerd solves these issues. It is a transparent proxy that adds:
	* service discovery
	* routing (latency aware LB)
	* shift traffic for canary deployments
	* failures (reties, deadlines, circuit breaking)
	* visibilityL: using web UI
* we will run linkerd on our nodes (DaemonSet)
* we will have 2 hello services and 2 world services
* linkerd is exposed with a LB 
* client => LB => linkerd(node2) => hello(node2) => world(node2)
* if world(node2) is down linklerd goes to linkerd(node1) => world(node1) transparently
* how do pods find linkerd??? nslookup : noDENAME where nodename is an env var. each node has a linkerd port listening
* hello needs to pass the linkerd proxy to reach world pod. this is  done in YAML config setting an http_proxy env var on node:4140 port

### Lecture 29 - Demo: Linkerd

* we spin a cluster on AWS (2 mediums 1 small)
* we ssh in vagrant
* linkerd works with http reqs through services (Inout)
* we go to advanced-kubernetes-course/linkerd and apply linkerd.yml. it creates the daemon set, an lb and a configmap
* it does transformations, routing in acomplex config map
* the hello-world.yml starts the hello and world services going through the proxy
* we follow README.md instructions `INGRESS_LB=$(kubectl get svc l5d -o jsonpath="{.status.loadBalancer.ingress[0].*}")`  and `echo http://$INGRESS_LB:9990`. we get the LB endpoint and launch it in browser
* we look at outgoing dashboard and apply the hellow-world.yml
* we want to contact the app `http_proxy=$INGRESS_LB:4140 curl -s http://hello` we hit multiple times to create traffic
* we apply to see grafana
```
kubectl apply -f linkerd-viz.yml
VIZ_INGRESS_LB=$(kubectl get svc linkerd-viz -o jsonpath="{.status.loadBalancer.ingress[0].*}")
echo http://$VIZ_INGRESS_LB
```

## Section 10 - Federation

### Lecture 30 - Introduction to Federation

* Federation can be used to manage multiple clusters
* we can sync resources across clusters. the same deployment version will run on cluster A and on cluster B
* we can do cross cluster discovery. on DNS record / virtual IP (VIP) for a resource spanning multiple clusters
* this can help achieve HA (High Availability) spreading load across clusters, enabling failover between clusters
* reasons to run multiple clusters:
	* Lower latency for customers (bringing app geographically closer to customer)
	* Fault isolation when physical hardware fails
	* Scale beyond cluster limits.
	* Run clusters in hybrid cloud env (on-premise - failover to cloud)
* Since kubernetes v1.5 we can use a tool called kubefed to administer our federated clusters
	* add/remove clusters
	* deploy a K8s federation control pane

### Lecture 31 - Demo: Federation with kops

* we go to vagrant vm on /advanced-kubernetes-course/federation
* we look at README for instruction (OMG!)
* we `$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)` to see current version. its 1.13.1
* we download an old version of k8s `curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.7.2/kubernetes-client-linux-amd64.tar.gz`
* chosing latest version might cause issues as kops might have not catch up
* we move it to local bin `sudo mv kubernetes/client/bin/* /usr/local/bin/` and make them executable `sudo chmod +x /usr/local/bin/kube*`
* we create 2 clusters in different zones
```
 kops create cluster --name=k8s.agileng.io --state=s3://kops-state-4213432 --zones=eu-central-1a --node-count=2 --node-size=t2.small --master-size=t2.small --dns-zone=k8s.agileng.io
kops create cluster --name=k8s-2.agileng.io --state=s3://kops-state-4213432 --zones=eu-central-1b --node-count=2 --node-size=t2.small --master-size=t2.small --dns-zone=k8s-2.agileng.io
kops update cluster k8s.agileng.io --yes --state=s3://kops-state-4213432
kops update cluster k8s-2.agileng.io --yes --state=s3://kops-state-4213432
```
* we need to add a second hosted zone and link it to our DNS provider
* its good to say to Route53 to create a federated namespace as kubefed can create hosted zones for us
* we need to play with IAM to allow k8s.agileng.io to create records on federated namespace
* kops creates IAM policies only for cluster namespaces
* in IAM roles nodes.k8s.agileng.io i edit the policy and got o JSON in route53 section adding a new section ofr a new hosted zone adding the ID of the federated . this is not a permanent solution . it will be overriten with kops edit and update
* to see both clusters i run `kubectll config get-context` and `kubectl config use-context k8s.agileng.io`
* once clusters have started we can initialize federations with `kubefed init federated --host-cluster-context=k8s.agileng.io --dns-provider="aws-route53" --dns-zone-name="federated.agileng.io.`
* thingsget started. it creates a PVC for etcd data
* the ELB created is a service of the primary cluster.
* if i run get-contexts i have 3 k8s k8s-2 and federated 
* i `kubectl config use-context federated` 
* i can now join both clusters
```
kubefed join kubernetes-2 --host-cluster-context=k8s.agileng.io --cluster-context=k8s-2.agileng.io
kubefed join kubernetes-1 --host-cluster-context=k8s.agileng.io --cluster-context=k8s.agileng.io
kubectl create namespace default --context=federated
```
* i can now use --context=federated in my kubectl commands so that my staate change will apply to the joined federatedcluster
* pods and services are distributed along clusters. zone rules apply
* in federated hostname (route53) cnames are added automatically as nodes and masters need to access it
* in alias for ELB i can add failover for second elb

## Section 11 - Monitoring

### Lecture 32 - Introduction to Prometheus

* Prometheus is a n open source monitoring and alerting tool
* We can compare it to the heapster + grafana setup from first course. Prometheus can do a lot more
* Built in Soundcloud, standalone and opensource project
* Member of Cloud native Computing FOundation since 2016
* Prometheus provides a multi-dimensional data model
	* Time series identified by metric name and key/val pair
* flexible query lang
* no distributed storage necessary
* metric collection through a http pull model
* push is supproted with a gateway
* service discovery supported
* Web UI (dashboard) with graphingcap abilities
* in our demo we will have 4 services. 2 kube-system and 1 app. 
* ServiceMonitor stands between the services and Prometheus. they are uses per service to inject metrics
* a Prometheus Operator will be deployed  to deploy resources that monitor our servcices  and deploys and manages a StatefulSet (Prometheus Server)

### Lecture 33 - Demo: Prometheus

* we need a cluster with 4gb nodes and 2gb master
* we go to advanced-kubernetes-course/prometheus
* we look at  README
* we create rbac.yml as prometheus nees serviceaccount with priviledges
* we create prometheus.yml. it has the deployment of prometheus-operator allowing to use prometheus resources i k8s. also a service (LB)
* prometheus-resource.yml uses a CRD (Prometheus)
* we create kubernetes-monitoring.yml create services to monitor and service monitors
* we create the example-app.yml just a simple webapp
* we check svc. to get LB link