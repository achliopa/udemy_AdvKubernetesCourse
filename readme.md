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

### Lecture 10 - Demo: OIDC with Auth0