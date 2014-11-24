# Continuous Integration and Performance Testing Boilerplate 

This repository provides a boilerplate Jenkins setup to run a full CI pipeline <br/> ``` build->test->integration_test->deploy->integration_test->perf_test ```.
 
The example setup builds & tests and deploys [Java REST Example App](https://github.com/hostirosti/java-rest-example-appengine) on Google App Engine.

## Jenkins CI Boilerplate Setup
We will use Google Container Engine for our demonstration setup.
It integrates with GitHub to automatically trigger the build & test pipeline.
The status(failed|unstable|success) of the build & test pipeline is reported back to GitHub.

### Prerequisites
 - Google Cloud Account with billing enabled (https://cloud.google.com/)
 - Google Cloud SDK installed on your machine
 - 2nd GitHub account for Jenkins Bot

### Create Google Cloud Project and setup Google Container Engine Cluster
 - Go to https://console.developers.google.com and follow the Google documentation how to create a project
 - Select your project and enable the Container Engine API. **[TODO] add more information**

A Google Container Cluster consits of [services](https://cloud.google.com/container-engine/docs/services/), 
[replication controllers](https://cloud.google.com/container-engine/docs/replicationcontrollers/) and 
[pods](https://cloud.google.com/container-engine/docs/pods/).

#### Create the Cluster
The example cluster ```ci-example-cluster``` uses the minimal # of instances (1 master + 1 node) of type ```n1-standard-1``` in zone ```us-central1-a```.

For ease of use set environment variables for *zone* and *cluster name*:
```
$ export ZONE=us-central1-a
$ export CLUSTER=ci-example-cluster
```

To create the cluster run:
```
$ sudo gcloud preview container clusters create $CLUSTER \
	--num-nodes 1 \
	--machine-type n1-standard-1 \
	--zone $ZONE
```

#### Prepare Container Host (cluster-node-1)

Login in to cluster-node-1 and create jenkins dir:
``` 
$ sudo gcloud compute ssh --zone $ZONE k8s-$CLUSTER-node-1
cluster-node-1 $ mkdir /opt/jenkins && chmod a+w /opt/jenkins
```

#### Create the Jenkins Pod
The pod configuration file [jenkins-master-pod.json](jenkins-master-pod.json) describes which docker container to start, which volumes to mount and which ports to map.
The Jenkins master pod starts a custom docker image [hostrosti/jenkins-gcloud-sdk](https://registry.hub.docker.com/u/hostirosti/jenkins-gcloud-sdk/) that has the lastest Jenkins and GCloud SDK installed, mounts the just created folder ```/opt/jenkins``` into the docker container under ```/var/jenkins_home``` and maps port ```8080``` to port ```8080``` on the host system.
```
$ sudo gcloud preview container pods create jenkins-master-pod \
	--cluster-name=$CLUSTER \
	--zone=$ZONE \
	--config-file=jenkins-master-pod.json
```

Wait for the pod to be started (this may take a couple minutes).
```
$ sudo gcloud preview container pods list --cluster-name=$CLUSTER --zone=$ZONE
ID                   Image(s)                               Host                                                                  Labels                    Status
----------           ----------                             ----------                                                            ----------                ----------
jenkins-master-pod   hostirosti/jenkins-gcloud-sdk:latest   k8s-ci-example-cluster-node-1.c.ci-perftest.internal/130.211.xxx.xxx  name=jenkins-master-pod   Waiting

... a couple minutes later ...

$ sudo gcloud preview container pods list --cluster-name=$CLUSTER --zone=$ZONE
ID                   Image(s)                               Host                                                                   Labels                    Status
----------           ----------                             ----------                                                             ----------                ----------
jenkins-master-pod   hostirosti/jenkins-gcloud-sdk:latest   k8s-ci-example-cluster-node-1.c.ci-perftest.internal/130.211.xxx.xxx   name=jenkins-master-pod   Running
```


#### Open the Firewall Port
To make Jenkins accessible from outside the cluster we need to open port ```8080```
```
$ sudo gcloud compute firewall-rules create $CLUSTER-node-8080 \
	--allow=tcp:8080 \
	--target-tags k8s-$CLUSTER-node
```

Your Jenkins should now be accessible under the up that is listed for your node when you get the pod list ```130.211.xxx.xxx```.
Check it at http://130.211.xxx.xxx:8080/


#### Configure your Jenkins Docker Image
We're almost done :) To be able to deploy to Google App Engine it is neccessary to authenticate the gcloud cmd tool on the docker container.

Login to your ```cluster-node-1``` and find your docker container and login.
```
$ sudo gcloud compute ssh --zone $ZONE k8s-$CLUSTER-node-1
cluster-node-1 $ docker ps
CONTAINER ID        IMAGE                                  COMMAND                ...
6b645b8b6131        hostirosti/jenkins-gcloud-sdk:latest   "/usr/local/bin/jenk   ...

cluster-node-1 $ docker exec -it <container-id> bash
jenkins@jenkins-master-pod:/$ glcoud auth login

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

You are now logged in as [ci-example-cluster@gmail.com].
Your current project is [ci-example-cluster].  You can change this setting by running:
  $ gcloud config set project PROJECT
 ```

Copy the displayed url in your prefered browser, authenticate the Google Cloud SDK with your gmail account and copy&paste the authentication token.

The last step on your Jenkins docker image is to add the example Jenkins configuration.
```
jenkins@jenkins-master-pod:/$ cd /var/jenkins_home && curl -o jenkins-config.zip \
	https://codeload.github.com/hostirosti/ci-boilerplate/zip/master

jenkins@jenkins-master-pod:/$ unzip jenkins-config.zip && \ 
	mv ci-boilerplate-master/jenkins-configuration/* . && \
	rm -rf ci-boilerplate-master jenkins-config.zip
```

After loading all configs into your ```jenkins_home``` you need to trigger a reload of the configuration.
Go to *Manage Jenkins -> Reload Configuration from Disk*.

[Temporary Hack] Currently there seems to be a problem with picking up the configuration for the Maven Installations.
If you encounter an build error with ```ERROR: A Maven installation needs to be available...``` please set the maven installations under
*Manage Jenkins -> Configure System* as follows
![Jenkins Config Maven Installations](/images/jenkins/jenkins_maven_settings.png)


### Configure GitHub Repository, Jenkins and GitHub PullRequest Plugin

#### Create/Fork GitHub repository and link to Jenkins
You can use your own application repository that you would like to connect to the CI environment or 
start from a fork of [hostirosti/java-rest-example-appengine](https://github.com/hostirosti/java-rest-example-appengine).

The preinstalled Jenkins jobs are configured for the example Java app and only need an adjustment of the GitHub repository url.
Make sure you change the url in the first 3 jobs (01,02,03).
![Set Git url to your fork](/images/git_url_change.png)

Generate an applications access token for your Jenkins GitHub user (Bot user).
Login on GitHub with your Bot user and go to *Settings -> Applications*. Click on *Generate new token*.
Select scopes ```repo``` and ```repo:status```
![Create Access Token for Jenkins Bot GitHub User](/images/generate_new_access_token.png)

You'll get an access token that you have to add in *Jenkins -> Global Configuration*
![Generated Application Access Token](/images/generated_access_token.png)

Go to Manage ```Jenkins -> Configure System``` and search for ```GitHub Pull Request Builder``` and add your just generated access token.
Also add your primary GitHub (not the Bot account) to the admin list.
![Add Access Token under GitHub Pull Requester Builder](/images/gprb_access_token.png)


#### Setup the webhook on your GitHub repository
Login with your main GitHub account.
Go to the repository settings of the forked Java REST Example repository *Settings -> Webhooks & Services -> Add Webhook*. <br/>
For '*Payload URL*' enter ```http://NODE_IP:8080/ghprbhook/``` and for '*Content type*' select ```application/x-www-form-urlencoded```. <br/>
Under '*Which events would you like to trigger this webhook*' select '*Let me select individual events.*' and choose '*Pull Request*' and '*Issue comment*' for this example.
Make sure to click on '*Add Webhook*' to save your settings.
![Repository Webhook Settings](/images/webhook_settings.png)

### Test the setup
If you forked the example Java App and adjusted the repository url in the Jenkins jobs you can test your setup by:
```
$ git clone git@github.com:<your_github_user>/java-rest-example-appengine
$ cd java-rest-example-appengine
$ git checkout -b test_branch
<make a code change>
$ git commit -a -m "Testing the setup"
$ git push -u origin test_branch
```
Create a Pull Request from ```test_branch``` onto ```master``` and comment on the created Pull request '*Jenkins it's your turn*'.
Watch the magic on your Jenkins :)


## Example Build&Test pipeline in more detail
If the build and tests are successful the app will be deployed to Google App Engine and triggers a 
performance test Jenkins job. 


