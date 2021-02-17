## Lab 1 - Go Microservices

Now that you have your environment provisioned and the prerequisites fulfilled, it is time to start working on the labs.

## Application Overview

You will be deploying a ratings application on Azure Red Hat OpenShift.

![Application diagram](../media/app-overview.png)

The application consists of 3 components:

| Component                                          | Link                                                               |
|----------------------------------------------------|--------------------------------------------------------------------|
| A public facing API `rating-api`                   | [GitHub repo](https://github.com/microsoft/rating-api)             |
| A public facing web frontend `rating-web`          | [GitHub repo](https://github.com/microsoft/rating-web)             |
| A MongoDB with pre-loaded data                     | [Data](https://github.com/microsoft/rating-api/raw/master/data.tar.gz)   |

Once you're done, you'll have an experience similar to the below.

![Application](../media/app-overview-1.png)
![Application](../media/app-overview-2.png)
![Application](../media/app-overview-3.png)

## Connect to the cluster

You can log into the cluster using the `kubeadmin` user.  

Now, initialise the values for Resource group name and Cluster name. You can find the Resource group name and Cluster name in lab environment details page. Replace the values and run the below commands:

```
export RESOURCEGROUP=<Resource_group_name>
export CLUSTER=<Cluster_name>
```

Run the following command to find the password for the `kubeadmin` user.

```azurecli-interactive
az aro list-credentials \
  --name $CLUSTER \
  --resource-group $RESOURCEGROUP
```

The following example output shows the password will be in `kubeadminPassword`.

```json
{
  "kubeadminPassword": "<generated password>",
  "kubeadminUsername": "kubeadmin"
}
```

Save these secrets, you are going to use them to connect to the Web Portal

## Create Project

### Login to the web console



Each Azure Red Hat OpenShift cluster has a public hostname that hosts the OpenShift Web Console.

You can use command `az aro list` to list the clusters in your current Azure subscription.

```sh
az aro list -o table
```

The cluster web console's URL will be listed. Open that link in new browser tab and login with the `kubeadmin` user and password retrieved earlier.

After logging in, you should be able to see the Azure Red Hat OpenShift Web Console.

![Azure Red Hat OpenShift Web Console](../media/openshift-webconsole.png)



### Retrieve the login command and token



> **Note** Make sure you complete the [prerequisites](#prereq) to install the OpenShift CLI on the Azure Cloud Shell.

Once you're logged into the Web Console, click on the username on the top right, then click **Copy login command**.

![Copy login command](../media/login-command.png)

Open the [Azure Cloud Shell](https://shell.azure.com) and paste the login command. You should be able to connect to the cluster.

![Login through the cloud shell](../media/oc-login-cloudshell.png)



### Create a project



A project allows a community of users to organize and manage their content in isolation from other communities.

```sh
oc new-project workshop
```

![Create new project](../media/oc-newproject.png)



> **Resources**

> * [ARO Documentation - Getting started with the CLI](https://docs.openshift.com/aro/4/cli_reference/openshift_cli/getting-started-cli.html)
> * [ARO Documentation - Projects](https://docs.openshift.com/aro/4/applications/projects/working-with-projects.html)

## Deploy MongoDB

### Create mongoDB from template


Azure Red Hat OpenShift provides a container image and template to make creating a new MongoDB database service easy. The template provides parameter fields to define all the mandatory environment variables (user, password, database name, etc) with predefined defaults including auto-generation of password values. It will also define both a deployment configuration and a service.

There are two templates available:

* `mongodb-ephemeral` is for development/testing purposes only because it uses ephemeral storage for the database content. This means that if the database pod is restarted for any reason, such as the pod being moved to another node or the deployment configuration being updated and triggering a redeploy, all data will be lost.

* `mongodb-persistent` uses a persistent volume store for the database data which means the data will survive a pod restart. Using persistent volumes requires a persistent volume pool be defined in the Azure Red Hat OpenShift deployment.

Run the below commands to download `mongodb-persistent` template and create the template in openshift namespace :

```
wget https://raw.githubusercontent.com/openshift/origin/master/examples/db-templates/mongodb-persistent-template.json

oc create -f mongodb-persistent-template.json -n openshift
```

> **Hint** You can retrieve a list of templates using the command below. The templates are preinstalled in the `openshift` namespace.
> ```sh
> oc get templates -n openshift
> ```

Create a mongoDB deployment using the `mongodb-persistent` template that you have created in openshift namespace. You're passing in the values to be replaced (username, password and database) which generates a YAML/JSON file. You then pipe it to the `oc create` command.

```sh
oc process openshift//mongodb-persistent \
    -p MONGODB_USER=ratingsuser \
    -p MONGODB_PASSWORD=ratingspassword \
    -p MONGODB_DATABASE=ratingsdb \
    -p MONGODB_ADMIN_PASSWORD=ratingspassword | oc create -f -
```

If you now head back to the web console, and switch to the **workshop** project, you should see a new deployment for mongoDB.

![MongoDB deployment](../media/mongodb-overview.png)



### Verify if the mongoDB pod was created successfully



Run the `oc get all` command to view the status of the new application and verify if the deployment of the mongoDB template was successful.

```sh
oc get all
```

![oc status](../media/oc-status-mongodb.png)



### Retrieve mongoDB service hostname



Find the mongoDB service.

```sh
oc get svc mongodb
```

![oc get svc](../media/oc-get-svc-mongo.png)

The service will be accessible at the following DNS name: `mongodb.workshop.svc.cluster.local` which is formed of `[service name].[project name].svc.cluster.local`. This resolves only within the cluster.

You can also retrieve this from the web console. You'll need this hostname to configure the `rating-api`.

![MongoDB service in the Web Console](../media/mongo-svc-webconsole.png)



> **Resources**
> * [ARO Documentation - Templates](https://docs.openshift.com/aro/4/openshift_images/using-templates.html)

## Deploy Ratings API

The `rating-api` is a NodeJS application that connects to mongoDB to retrieve and rate items. Below are some of the details that you'll need to deploy this.

- `rating-api` on GitHub: <https://github.com/microsoft/rating-api>
- The container exposes port 8080
- MongoDB connection is configured using an environment variable called `MONGODB_URI`

### Fork the application to your own GitHub repository

To be able to setup CI/CD webhooks, you'll need to fork the application into your personal GitHub repository.

<a class="github-button" href="https://github.com/microsoft/rating-api/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork microsoft/rating-api on GitHub">Fork</a>

### Use the OpenShift CLI to deploy the `rating-api`

> **Note** You're going to be using [source-to-image (S2I)](#source-to-image-s2i) as a build strategy.



```sh
oc new-app https://github.com/<your GitHub username>/rating-api --strategy=source
```

![Create rating-api using oc cli](../media/oc-newapp-ratingapi.png)



### Configure the required environment variables



Create the `MONGODB_URI` environment variable. This URI should look like `mongodb://[username]:[password]@[endpoint]:27017/ratingsdb`. You'll need to replace the `[usernaame]` and `[password]` with the ones you used when creating the database. You'll also need to replace the `[endpoint]` with the hostname acquired in the previous step

Hit **Save** when done.

![Create a MONGODB_URI environment variable](../media/rating-api-envvars.png)

It can also be done with CLI

```
oc set env deploy/rating-api MONGODB_URI=mongodb://ratingsuser:ratingspassword@mongodb.workshop.svc.cluster.local:27017/ratingsdb
```



### Verify that the service is running



If you navigate to the logs of the `rating-api` deployment, you should see a log message confirming the code can successfully connect to the mongoDB.
For that, in the deployment's details screen, click on *Pods* tab, then on one of the pods

![Verify mongoDB connection](../media/rating-api-working.png)



### Retrieve `rating-api` service hostname



Find the `rating-api` service.

```sh
oc get svc rating-api
```

The service will be accessible at the following DNS name over port 8080: `rating-api.workshop.svc.cluster.local:8080` which is formed of `[service name].[project name].svc.cluster.local`. This resolves only within the cluster.



### Setup GitHub webhook

To trigger S2I builds when you push code into your GitHib repo, you'll need to setup the GitHub webhook.



Retrieve the GitHub webhook trigger secret. You'll need use this secret in the GitHub webhook URL.

```sh
oc get bc/rating-api -o=jsonpath='{.spec.triggers..github.secret}'
```

You'll get back something similar to the below. Make note the secret key in the red box as you'll need it in a few steps.

![Rating API GitHub trigger secret](../media/rating-api-github-secret.png)

Retrieve the GitHub webhook trigger URL from the build configuration.

```sh
oc describe bc/rating-api
```

![Rating API GitHub trigger url](../media/rating-api-github-webhook-url.png)

Replace the `<secret>` placeholder with the secret you retrieved in the previous step to have a URL similar to `https://api.otyvsnz3.eastus.aroapp.io:6443/apis/build.openshift.io/v1/namespaces/workshop/buildconfigs/rating-api/webhooks/SECRETSTRING/github`. You'll use this URL to setup the webhook on your GitHub repository.

In your GitHub repository, select **Add Webhook** from **Settings** → **Webhooks**.

Paste the URL output (similar to above) into the Payload URL field.

Change the Content Type from GitHub’s default **application/x-www-form-urlencoded** to **application/json**.

Click **Add webhook**.

![GitHub add webhook](../media/rating-api-github-addwebhook.png)

You should see a message from GitHub stating that your webhook was successfully configured.

Now, whenever you push a change to your GitHub repository, a new build will automatically start, and upon a successful build a new deployment will start.



> **Resources**
> * [ARO Documentation - Triggering builds](https://docs.openshift.com/aro/4/builds/triggering-builds-build-hooks.html)

## Deploy Ratings frontend

The `rating-web` is a NodeJS application that connects to the `rating-api`. Below are some of the details that you'll need to deploy this.

- `rating-web` on GitHub: <https://github.com/microsoft/rating-web>
- The container exposes port 8080
- The web app connects to the API over the internal cluster DNS, using a proxy through an environment variable named `API`

### Fork the application to your own GitHub repository

To be able to setup CI/CD webhooks, you'll need to fork the application into your personal GitHub repository.

<a class="github-button" href="https://github.com/microsoft/rating-web/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork microsoft/rating-web on GitHub">Fork</a>

### Use the OpenShift CLI to deploy the `rating-web`

> **Note** You're going to be using [source-to-image (S2I)](#source-to-image-s2i) as a build strategy.



```sh
oc new-app https://github.com/<your GitHub username>/rating-web --strategy=source
```

![Create rating-web using oc cli](../media/oc-newapp-ratingweb.png)



### Configure the required environment variables



Create the `API` environment variable for `rating-web` Deployment Config. The value of this variable is going to be the hostname/port of the `rating-api` service.

Instead of setting the environment variable through the Azure Red Hat OpenShift Web Console, you can set it through the OpenShift CLI.

```sh
oc set env deploy rating-web API=http://rating-api:8080

```



### Expose the `rating-web` service using a Route



Expose the service.

```sh
oc expose svc/rating-web
```

Find out the created route hostname

```sh
oc get route rating-web
```

You should get a response similar to the below.

![Retrieve the created route](../media/oc-get-route.png)

Notice the fully qualified domain name (FQDN) is comprised of the application name and project name by default. The remainder of the FQDN, the subdomain, is your Azure Red Hat OpenShift cluster specific apps subdomain.



### Try the service



Open the hostname in your browser, you should see the rating app page. Play around, submit a few votes and check the leaderboard.

![rating-web homepage](../media/rating-web-homepage.png)



### Setup GitHub webhook

To trigger S2I builds when you push code into your GitHib repo, you'll need to setup the GitHub webhook.



Retrieve the GitHub webhook trigger secret. You'll need use this secret in the GitHub webhook URL.

```sh
oc get bc/rating-web -o=jsonpath='{.spec.triggers..github.secret}'
```

You'll get back something similar to the below. Make note the secret key in the red box as you'll need it in a few steps.

![Rating Web GitHub trigger secret](../media/rating-web-github-secret.png)

Retrieve the GitHub webhook trigger URL from the build configuration.

```sh
oc describe bc/rating-web
```

![Rating Web GitHub trigger url](../media/rating-web-github-webhook-url.png)

Replace the `<secret>` placeholder with the secret you retrieved in the previous step to have a URL similar to `https://api.otyvsnz3.eastus.aroapp.io:6443/apis/build.openshift.io/v1/namespaces/workshop/buildconfigs/rating-web/webhooks/SECRETSTRING/github`. You'll use this URL to setup the webhook on your GitHub repository.

In your GitHub repository, select **Add Webhook** from **Settings** → **Webhooks**.

Paste the URL output (similar to above) into the Payload URL field.

Change the Content Type from GitHub’s default **application/x-www-form-urlencoded** to **application/json**.

Click **Add webhook**.

![GitHub add webhook](../media/rating-web-github-addwebhook.png)

You should see a message from GitHub stating that your webhook was successfully configured.

Now, whenever you push a change to your GitHub repository, a new build will automatically start, and upon a successful build a new deployment will start.



### Make a change to the website app and see the rolling update



Go to the `https://github.com/<your GitHub username>/rating-web/blob/master/src/App.vue` file in your repository on GitHub.

Edit the file, and change the `background-color: #999;` line to be `background-color: #0071c5`.

Commit the changes to the file into the `master` branch.

![GitHub edit app](../media/rating-web-editcolor.png)

Immediately, go to the **Builds** tab in the OpenShift Web Console. You'll see a new build queued up which was triggered by the push. Once this is done, it will trigger a new deployment and you should see the new website color updated.

![Webhook build](../media/rating-web-cicd-build.png)

![New rating website](../media/rating-web-newcolor.png)



> **Resources**
> * [ARO Documentation - Triggering builds](https://docs.openshift.com/aro/4/builds/triggering-builds-build-hooks.html)

## Create Network Policy

Now that you have the application working, it is time to apply some security hardening. You'll use [network policies](https://docs.openshift.com/aro/4/networking/network_policy/about-network-policy.html) to restrict communication to the `rating-api`.

### Switch to the Cluster Console



Switch to the Administrator console.
![Switch to the Administrator console](../media/switch-to-admin-console.png)

Make sure you're in the **workshop** project, expand **Networking** and click **Create Network Policy**.
![Cluster console page](../media/cluster-console.png)



### Create network policy



You will create a policy that applies to any pod matching the `app=rating-api` label. The policy will allow ingress only from pods matching the `app=rating-web` label.

Use the YAML below in the editor, and make sure you're targeting the **workshop** project.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-web
  namespace: workshop
spec:
  podSelector:
    matchLabels:
      app: rating-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: rating-web
```

![Create network policy](../media/create-networkpolicy.png)

Click **Create**.



> **Resources**
> * [ARO Documentation - Managing Networking with Network Policy](https://docs.openshift.com/aro/4/networking/network_policy/creating-network-policy.html)
