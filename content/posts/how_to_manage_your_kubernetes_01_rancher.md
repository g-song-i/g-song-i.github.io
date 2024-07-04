+++
title="How to manage your kubernetes cluster 01: Rancher"
date=2024-07-04

[taxonomies]
categories = ["Managing Clusters", "Kubernetes", "Rancher", "SUSE"]
tags = ["post"]
+++

# Table of Contents
[Overview](#overview) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Install Rancher with Docker](#install-rancher-with-docker) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Dive into Rancher](#dive-into-rancher) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Login](#login) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Importing your existing cluster](#importing-your-existing-cluster) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Explore Dashboard](#explore-dashboard) <br/>
[More...](#more) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Logging in Rancher](#logging-in-rancher) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Setting Up Alerts](#setting-up-alerts) <br/>
[When Is Using Rancher a Good Choice?](#when-is-using-rancher-a-good-choice) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[User-Friendliness for Non-Terminal Users](#user-friendliness-for-non-terminal-users) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Comprehensive Management from a Single Pane](#comprehensive-management-from-a-single-pane) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Extensive Catalog of Applications](#extensive-catalog-of-applications) <br/>

# Overview

Do you want to manage your Kubernetes cluster? There are several ways to manage it such as using the kubectl CLI if you are familiar with the terminal interface, Rancher from SUSE if you prefer managing through a console, and k9s if you want a more stylish approach!

In this post, I will explain how to install Rancher for managing Kubernetes and discuss when it is a good choice.


## Install Rancher with Docker

Note that this method of installation is just for testing and demonstration purposes as described at [Rancher documentation](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade#docker-install). You should choose another method if it is for production use.

Here I have a single docker-compose file to set up and tear down the Rancher container:

```
version: '3'

services:
  rancher:
    image: rancher/rancher:v2.8-head
    container_name: rancher_test
    restart: always
    ports:
    - "80:80/tcp"
    - "443:443/tcp"
    volumes:
    - "./rancher-data:/var/lib/rancher"
    environment:
    - CATTLE_BOOTSTRAP_PASSWORD=1234
    privileged: true

```

Now you just need to run the command (you might need "sudo" privileges):

```bash
$ docker-compose up
Creating network "rancher_default" with the default driver
Pulling rancher (rancher/rancher:v2.8-head)...
v2.8-head: Pulling from rancher/rancher
088fce18aff4: Pull complete
8c6a13d6a83e: Pull complete
378e6ec2cbbc: Pull complete
...
```

## Dive into Rancher
### Login
Now you can access the website via "http". Then you might see the login screen. Since we already defined the bootstrap password (CATTLE_BOOTSTRAP_PASSWORD) in the docker-compose file, enter "1234".

![RancherBootstrapLogin](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/7dd8709b-29d3-42a1-9185-a923d0cb5a37) <br/>

### Importing your existing cluster
Let's import your existing cluster. To import the cluster into Rancher, you must have "cluster-admin" privileges. You can apply a clusterrolebinding as follows:
```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user [USER_ACCOUNT]
```

You can check which account you are using with the command (user in context):

```
kubectl config view --minify
(or) kubectl config current-context
```

Now click "Import Existing", 
![ImportingClusterButton](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/1f96795c-4e36-46ea-8f4c-2657949ddd59) <br/>

Write the cluster's name, and press the create button. 
![CreateCluster](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/14c6563b-29d4-4786-9649-3547f586adc1) <br/>


Then you might see some command lines to apply Rancher's YAML to create some pods for importing. I tried the first one `kubectl apply -f https://~.yaml`, but I encountered the message `Unable to connect to the server: tls: failed to verify certificate: x509: certificate signed by unknown authority`, I tried the second line `curl --insecure --sfL ... | kubectl apply -f -`, and it worked for me.

You can enter the command `kubectl get pods -A` just to see what pods are created for this work. 

```
$ kubectl get pods -A
NAMESPACE       NAME                                       READY   STATUS              RESTARTS   AGE
cattle-system   cattle-cluster-agent-845554d4f8-qnksl      0/1     ContainerCreating   0          12s
```

And now you can explore your cluster and their resources on the Rancher dashboard!
![dashborad](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/6dc20e0f-03f3-4b68-8760-505c1957fbfe)

### Explore Dashboard
From here, you can see useful cluster information to inspect and debug your cluster. Rancher's dashboard offers a comprehensive overview of your cluster's health, resource usage, and more. You can utilize the dashboard to manage workloads, configure network policies, and set up service discovery.

## More...
### Logging in Rancher
Rancher enhances the management of Kubernetes clusters by providing comprehensive logging capabilities that are crucial for monitoring the health and activities within your cluster. Here’s a brief overview of the logging features in Rancher:
* Centralized Logging
* Integration with Multiple Logging Backends
* Log Analysis and Troubleshooting

### Setting Up Alerts
Rancher allows you to set up [alerts](https://ranchermanager.docs.rancher.com/v2.0-v2.4/reference-guides/rancher-project-tools/project-alerts) for various events within your cluster. This feature helps in proactive monitoring and can alert administrators about critical issues before they impact your operations. Setting up alerts involves specifying the conditions and thresholds that trigger notifications.

By integrating these features into your daily operations, you can maintain a robust, efficient, and secure Kubernetes environment, ensuring your applications run smoothly and reliably.

# When Is Using Rancher a Good Choice?
While tools like kubectl and k9s offer powerful ways to interact with Kubernetes, each has its context where it excels. Rancher, in particular, stands out in several scenarios:

## User-Friendliness for Non-Terminal Users
One of the primary advantages of using Rancher is its user-friendly graphical interface. If you or your team are not familiar or comfortable with command-line interfaces, Rancher provides a more accessible entry point into managing Kubernetes. Its dashboard presents complex cluster information and operations in a visual format, making it easier to understand and manage without deep command-line expertise.

## Comprehensive Management from a Single Pane
Rancher offers a centralized management pane for all Kubernetes clusters. This is particularly useful for organizations managing multiple clusters, possibly across different environments (cloud and on-prem). Instead of switching between different terminal windows or scripts, Rancher gives you a unified overview and control point for deploying, managing, and scaling applications across all your clusters.

## Extensive Catalog of Applications
Rancher comes with a built-in catalog of Helm charts and other applications, which can be deployed with just a few clicks. This is particularly advantageous for teams looking to rapidly deploy and test new applications without going through the CLI-based setup of Helm or other package managers.