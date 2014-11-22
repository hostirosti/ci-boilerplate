# Continuous Integration and Performance Testing Boilerplate 

This repository contains a boilerplate Jenkins setup to run a full CI pipeline <br/>``` build->test->integration_test->deploy->integration_test->perf_test ```.
 
The boilerplate scripts refer to a Java app example that is deployed and tested on Google App Engine.

## Jenkins CI Boilerplate Setup
The example setup runs on Google's Container Engine.
It integrates with GitHub to automatically trigger the build&test pipeline.
The status(failed|unstable|success) of the build&test pipeline is reported back to GitHub.

### Prerequisites
 - Google Cloud Account with billing enabled (https://cloud.google.com/)
 - Google Cloud SDK installed on your machine

### Create Project and setup Google Container Engine Cluster
 - Go to https://console.developers.google.com and follow the Google documentation how to create a project
 - Select your project and enable the Container Engine API. **WIP**

A Google Container Cluster consits of services, replication controllers and pods. Please refer to the Google documentation for details.

#### Create the Cluster
The example cluster ```ci-example``` uses the minimal # of instances (1 master + 1 node) of type ```n1-standard-1``` in zone ```us-central1-a```.

For ease of use you can set environment variables for *zone* and *cluster name*:
```
$ export ZONE=us-central1-a
$ export CLUSTER=ci-example
```

To create the cluster execute:
```
$ gcloud preview clusters create $CLUSTER \
	--num-nodes 1 \
	--machine-type n1-standard-1 \
	--zone $ZONE
```

#### Prepare Container Host (cluster-node-1)

Login in to cluster-node-1 and create jenkins dir:
``` 
$ gcloud compute ssh --zone $ZONE xxx-ci-example-docker-cluster-node-1
cluster-node-1 $ sudo su -
cluster-node-1 $ mkdir /opt/jenkins
```

#### Create the Jenkins Pod
The pod configuration file [jenkins-master-pod.json](jenkins-master-pod.json) describes which docker container to start, which volumes to mount and which ports to map.
The Jenkins master pod starts a custom docker image [hostrosti/jenkins-gcloud-sdk](https://registry.hub.docker.com/u/hostirosti/jenkins-gcloud-sdk/) that has the lastest Jenkins and GCloud SDK installed, mounts the just created folder ```/opt/jenkins``` into the docker container under ```/var/jenkins_home``` and maps port ```8080``` to port ```8080``` on the host system.
```
$ gcloud preview container pods create jenkins-master-pod \
	--cluster-name=$CLUSTER \
	--zone=$ZONE \
	--config-file=jenkins-master-pod.json
```

#### Open the Firewall Port
To make Jenkins accessible from outside the cluster we need to open port ```8080```
```
$ gcloud compute firewall-rules create $CLUSTER-node-8080 \
	--allow=tcp:8080 \
	--target-tags k8s-$CLUSTER_NAME-node

```

#### Configure your Jenkins Docker image
To be able to deploy to Google App Engine it is neccessary to authenticate the gcloud cmd tool on the docker container.
Login to your ```cluster-node-1``` find and login to the docker container.
```
$ gcloud compute ssh --zone $ZONE xxx-ci-example-docker-cluster-node-1
cluster-node-1 $ sudo su -
cluster-node-1 $ docker ps
....
cluster-node-1 $ docker exec -it <container-id> bash
jenkins@jenkins-master-pod:~$ glcoud auth login

You are running on a GCE VM. It is recommended that you use
service accounts for authentication.

You can run:

  $ gcloud config set account ``ACCOUNT''

to switch accounts if necessary.

Your credentials may be visible to others with access to this
virtual machine. Are you sure you want to authenticate with
your personal account?

Do you want to continue (Y/n)?  Y

Go to the following link in your browser:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=...


Enter verification code: [authentication token]
Saved Application Default Credentials.

You are now logged in as [ci-example@gmail.com].
Your current project is [ci-example].  You can change this setting by running:
  $ gcloud config set project PROJECT
 ```

Copy the displayed url in your prefered browser, authenticate the Google Cloud SDK on your gmail account and copy&paste the authentication token.

Your docker instance is now ready to go on ```http://EXTERNAL_IP:8080```. To find out the ```EXTERNAL_IP``` of ```cluster-node-1``` execute:
```
$  gcloud compute instances list --zone $ZONE
NAME               ZONE          MACHINE_TYPE  INTERNAL_IP    EXTERNAL_IP    STATUS
xxx-cluster-master us-central1-a n1-standard-1 10.240.47.102  146.148.34.109 RUNNING
xxx-cluster-node-1 us-central1-a n1-standard-1 10.240.205.88  130.211.117.34 RUNNING
```

### Configure GitHub and Jenkins

#### Create repository and link to Jenkins
To start you may fork [hostirosti/java-rest-example-appengine](https://github.com/hostirosti/java-rest-example-appengine) which has setup ...


#### Setup webhook on your GitHub repository

Go to *Settings -> Webhooks & Services -> Add Webhook*. For *Payload URL* enter ```http://EXTERNAL_PI:8080/ghprbhook/``` and for content type select ```application/x-www-form-urlencoded```.
Under *Which events would you like to trigger this webhook* select *Let me select individual events.* and choose *Pull Request* and *Issue comment* for this example.




## Example Build&Test pipeline in more detail:
If the build and tests are successful the app will be deployed to e.g. Google App Engine and triggers a 
performance test Jenkins job. 

The performance test Jenkins Job starts a defined number of docker container that run the performance test. 
After they are finished, all performance test docker container will be stopped and the app will be shutdown.


