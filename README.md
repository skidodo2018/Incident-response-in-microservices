# Incident-response-in-microservices Project 6

Anyone who has ever been tasked with identifying the root cause of an incident by searching through huge volumes of logs knows what a painstaking process it is.
I remember having to sift through Splunk logs in order to identify the point of scan failure in my  experience as a vulnerability manager and it wasn't funny as I had to look in different directions.
Unfortunately, this process will get more difficult as applications become more complex and distributed.

## Project Objective
Demonstrate an alternative approach that uses the [Zebrium](https://www.zebrium.com/) machine learning (ML) platform to automatically find root cause in logs generated by an application deployed in [Amazon EKS](https://aws.amazon.com/eks/).

The project will cover:

- Installing the [Sock Shop](https://microservices-demo.github.io/) microservices demo app in an EKS cluster
- Installing Zebrium log collector
- Breaking the demo app (using a chaos engineering tool) and verifying that the Zebrium platform automatically finds the root cause

![](/images6/sockshop.png)

### Prerequisites

1. An active AWS account, click [here](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) to learn how to create one.
2. AWS CLI with the IAM user having admin permission or having all the permissions to execute the setup. Here's a guide detailing [how to install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and this guide details [how to create an IAM user with Admin permission](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html).
3. A free [Zebrium](http://www.zebrium.com/sign-up) trial account.

### Step 1. Create and configure an EKS cluster
We will assume that you already have an Amazon EKS cluster up and running. If not, you can follow this [link](https://www.eksworkshop.com/030_eksctl/) to get started with Amazon EKS cluster setup.

- Sign in to AWS and type EKS in the search bar.
- Click add cluster then select create.

![](/images6/EKS.png)

- Name the project and select a role for the EKS Cluster. Follow the link highlighted in the image below to create a role if none exists.

![](/images6/cluster-config.png)

- Leave other settings as default, click next and finally create.

- The status of the cluster will initially show "creating" and eventually become "active"
![](/images6/creatingcluster.png)
![](/images6/active-cluster.png)

- Next, add permissions to the cluster by creating a nodegroup.
Navigate to IAM services by typing same in the search bar and hitting "Enter", select roles.

![](/images6/iam.png)

- Click create role, select AWS service, then EC2 and "Next"

![](/images6/createrole.png)

- Add the following permissions and they should show up on the next page.

* AmazonEKSWorkerNodePolicy
* AmazonEKS_CNI_Policy
* AmazonEC2ContainerRegistryReadOnly

![](/images6/createnodegrp.png)

- Click create.
- Next, navigate to the Amazon EKS cluster previously created; Click the "Configuration" tab; select the "Compute" tab and then select "Add Node Group."

![](/images6/confignodegrp.png)

- Next, name the new node group and select the "Node IAM Role", select "Next", and then "Create".

![](/images6/nodegrp.png)

### Step 2. Create a Zebrium account and install the log collector

You’ll need a free Zebrium trial account (sign up [here](https://cloud.zebrium.com/auth/sign-up)). Create a new account, set your password and then advance to the Send Logs page.

![](/images6/zebrium.png)

**Important:** Do not install the log collector yet as we’re going to modify the install command!

![](/images6/logsetup.png)

1. Copy the Helm command from the Zebrium Send Logs page.

**Important:** You can just delete this part of the line:

```bash
zebrium.timezone=KUBERNETES_HOST_TIMEZONE,zebrium.deployment=YOUR_DEPLOYMENT_NAME
```

See the example below (make sure to substitute XXXX for your actual token).

```bash
# Install the Ze log collector. Copy the cmds from your browser & make the above changes. Make sure you use the correct token and URL

helm upgrade -i zlog-collector zlog-collector --namespace zebrium --create-namespace --repo https://raw.githubusercontent.com/zebrium/ze-kubernetes-collector/master/charts --set zebrium.collectorUrl=https://cloud-ingest.zebrium.com,zebrium.authToken=XXXX

```

![](/images6/logcollector.png)

2. The Zebrium UI should detect that logs are being sent within a minute or two. You’ll see a message like this:

![](/images6/firstlogs.png)

We’re now done with the Zebrium installation and setup. The machine learning will begin structuring and learning the patterns in the logs from your newly created K8s environment.

### Step 3. Install and fire up the Sock Shop demo app

[Sock Shop](https://microservices-demo.github.io/) is a really good demo microservices application as it simulates the key components of the user-facing part of an e-commerce website. It is built using Spring Boot, Go kit, and Node.js and is packaged in Docker containers. Visit [this GitHub page](https://github.com/microservices-demo/microservices-demo/blob/master/internal-docs/design.md) to learn more about the application design.

![](/images6/sockshop2.png)

A little bit later, we’re going to install and use the Litmus Chaos Engine to “break” the Sock Shop application. So, we are going to install Sock Shop using a YAML config file that contains annotations for the Litmus Chaos Engine.

Install Sock Shop from yaml. This version of the yaml has Litmus chaos annotations.

`$ kubectl create -f https://raw.githubusercontent.com/zebrium/zebrium-sockshop-demo/main/sock-shop-litmus-chaos.yaml`

![](/images6/sockshop-create.png)

Wait until all the pods are in a running state (this can take a few minutes):

```
# Check if everything has started - this takes a few minutes. Keep checking and don't move on until all pods are in a running state

kubectl get pods -n sock-shop
```

![](/images6/kuberunning.png)

When all the services are running, you can bring up the app in your browser. You will need to set up port forwarding and get the front-end IP address and port by running the command below. You should do this in a separate shell window.

```bash
#Get pod name of front-end

kubectl get pods -n sock-shop | grep front-end
front-end-77c5479795-f7wfk      1/1     Running   1          20h

#Run port forward in a new shell window
#Use pod name from the above command in place of XXX’s

kubectl port-forward front-end-XXXX-XXXX 8079:8079 -n sock-shop

Forwarding from 127.0.0.1:8079 -> 8079
Forwarding from [::1]:8079 -> 8079
Handling connection for 8079

#Open address:port in browser (127.0.0.1:8079 in example above)
```
![](/images6/front-end.png)
![](/images6/portforward.png)

Now open the ip_address:port from above (in this case: 127.0.0.1:8079) in a new browser tab. You should now be able to interact with the Sock Shop app in your browser and verify that it’s working correctly.

![](/images6/sockshop3.png)
![](/images6/sockshop4.png)

You can also go to CloudWatch in the AWS Console and visit the Resources page under Container Insights to verify that everything looks healthy. Details on how to do this can be found at the Amazon CloudWatch User Guide.

### Step 4. Install the Litmus Chaos Engine

We’re going to use the open-source Litmus chaos tool to run a chaos experiment that “breaks” the Sock Shop app. Install the required Litmus components using the following commands:

```bash
# Install Litmus and create an appropriate RBAC role for the pod-network-corruption test

helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/

helm upgrade -i litmus litmuschaos/litmus-core -n litmus --create-namespace

kubectl apply -f "https://hub.litmuschaos.io/api/chaos/1.13.6?file=charts/generic/experiments.yaml" -n sock-shop
```
![](/images6/genericexperiment.png)

```
# Setup service account with the appropriate RBAC to run the network corruption experiment

kubectl apply -f https://raw.githubusercontent.com/zebrium/zebrium-sockshop-demo/main/pod-network-corruption-rbac.yaml
```

![](/images6/podnet-corrption.png)

```
# Make a note of the time

date
```

### Step 5. Do something else for two hours!

This is a new EKS cluster and a new app and Zebrium account, so it’s important to give the machine learning a bit of time to learn the normal log patterns. We recommend waiting at least **two hours** before proceeding (you can wait longer if you like).

You can use this time to explore the Zebrium UI.

On the REPORTING page, you should see at least one sample root cause report.

![](/images6/zebriumrpt.png)

Select it and explore how to interact with Root Cause Reports! Make sure you try “peeking” and zooming into Related Events.

You might also see other real root cause reports. If you do, they are likely due to anomalous patterns in the logs that occur during the bring-up of the new environment (Zebrium sees them as being anomalous because it doesn’t have much history to go on at this stage). Again, feel free to poke around.

### Step 6. Run a network corruption chaos experiment to break the Sock Shop app.

Now that the ML has had a chance to get a baseline of the logs, we’re going to break the environment by running a Litmus network corruption chaos experiment.

The following command will start the network corruption experiment:

```bash
# Run the pod-network-corruption test
kubectl apply -f https://raw.githubusercontent.com/zebrium/zebrium-sockshop-demo/main/pod-network-corruption-chaos.yaml

# Make note  of the date
date
```

It will take a minute or so for the experiment to start. You can tell that it’s running when the pod-network-corruption-helper goes into a Running state.  Monitor its progress with kubectl (the -w option waits for output, so hit ^C once you see that everything is running):

`kubectl get pods -n sock-shop -w`

![](/images/getpods.png)

Once the chaos test has started running, you can go to the Sock Shop UI. You should still be able to navigate around and refresh the app but might notice some operations will fail (for example, if you add anything into the shopping cart, the number of items does not increment correctly). The chaos test will run for two minutes. Wait for it to complete before proceeding to the next step.

### Step 7. The results and how to interpret them

Please give the machine learning a few minutes to detect the problem (typically 2 to 10 minutes) and then refresh your browser window until you see one or more new root cause reports **(UI refresh is NOT automatic)**.

I tried the above procedure several times and saw really awesome results after each run. However, the actual root cause reports were different across runs. This is because of many factors, including the learning period, what events occurred while learning, the timing and order of the log lines while the experiment was running, other things happening on the system, and so on.

**Important: If you don’t see a relevant report, or the report doesn’t show you root cause indicators of the problem, read the section below,** “What should I do if there are no relevant root cause reports?”

Below are two examples and explanations of what I saw.

**Important**: We recommend that you read this section carefully as it will help you interpret the results when you try this on your own.

### Results example 1:

1. **Reporting page shows the first clue**

The Reporting page contains a summary list of all the root cause reports found by the machine learning. After running the chaos experiment, I saw this:

![](/images6/order-error.JPG)

There are three useful parts of the summary:

**Plain language NLP summary** – This is an experimental feature where we use the [GPT-3 language model](https://www.zebrium.com/blog/using-gpt-3-with-zebrium-for-plain-language-incident-root-cause-from-logs) to construct a summary of the report. In this case, the summary is not perfect, but it gives us some useful context about the problem. We see something about a Chaos Engine and an order not being created in the allotted time.

**Log type(s) and host(s)** – Here you can see the host and log types (front end, events, orders, and messages) that contain the events for this incident.

**One or two log “Hallmark” events** – The ML picks out one or two events that it thinks will characterize the problem. In this example, we see a chaos event as well as an exception about an order timeout. Just from this, we get a good sense of the problem.

2. **Viewing the report details shows the root cause and symptoms**

Clicking on the summary lets you drill down into the details of the root cause report. Initially, you see only the core events. The core events represent the cluster of correlated anomalies that the ML picks out. There are many factors that go into anomaly scoring, but the most important ones are events that are either “rare” or “bad” (events with high severity).

![](/images6/rptdetails.png)

As you can see, there are many root causes as to why the website is not functioning properly. Zebrium was able to find and different root causes with explanations.

**The End!**

Now that we project has concluded, don't forget to delete your cluster and node groups, so that you will not be charged for extensive usage.

