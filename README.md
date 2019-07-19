# Example Voting App

**Credit**: this repo is forked from https://github.com/yiliaomsft/example-voting-app which added steps to run on Azure App Servcie with Multi-container support.

**Original repo**: based on the Docker sample - https://github.com/dockersamples/example-voting-app 

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop). If you are on Mac or Windows, [Docker Compose](https://docs.docker.com/compose) will be automatically installed.

If you're using Windows, you must also [switch to Linux containers](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers).

On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

Run in this directory:

```sh
docker-compose up
```

This will build Docker images from the source code for `vote`, `result` and `worker`.

The app will be running at [http://localhost](http://localhost), and the results will be at [http://localhost/result](http://localhost/result).

To stop app and clean-up containers:

```sh
docker-compose down
```

## Run the app in Azure App Service (multi-container web app)

### Overview

Make sure you have the latest [Azure CLI installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli). 

Follow the steps in this [tutorial](https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-multi-container-app) to create:

* Azure Resource Group
* Linux App Service plan

Please ignore the steps for downloading the sample repo.

Once you have a Linux App Service plan, you can run the following CLI to create a Web App for this voting sample:

```sh
az webapp create --resource-group [group-name] --plan [service-plan] --name [app-name] --multicontainer-config-type "compose" --multicontainer-config-file "docker-compose-appservice.yml"
```

### Example steps

```sh
az group create --name votingapp --location "westus"

az appservice plan create --name votingapp --resource-group votingapp --sku S1 --is-linux

SUFFIX="$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 6 | head -n 1)"
echo "Your voting app suffix is: ${SUFFIX}"

APP_NAME="votingapp${SUFFIX}"

az webapp create --resource-group votingapp --plan votingapp --name "${APP_NAME}" --multicontainer-config-type "compose" --multicontainer-config-file "docker-compose-appservice.yml"

# Allow extra time to download all the Docker images for the first time onto the App Service plan instance
az webapp config appsettings set --resource-group votingapp --name "${APP_NAME}" --settings WEBSITES_CONTAINER_START_TIME_LIMIT="600"

# Retrieve the FQDN to paste in your browser
az webapp show -n "${APP_NAME}" -g votingapp --query "defaultHostName"

# Turn on diagnostic logs
az webapp log config --name "${APP_NAME}" --resource-group votingapp --docker-container-logging filesystem

# Stream logs to what deploy activity (If you don't see console logs immediately, check again in 30 seconds.)
az webapp log tail --name "${APP_NAME}" --resource-group votingapp
# Press CTRL+C to stop tailing logs
```

Wait for about a minute, you can then access the voting app at: [https://[appname].azurewebsites.net/](https://[appname].azurewebsites.net/), and the results will be at [https://[appname].azurewebsites.net/result](https://[appname].azurewebsites.net/result).

**Note**: The website may take a few mins to warm up since the Docker images must first be downloaded to the App Service Plan instance(s).  This can be minimised by using a [staging slot](https://docs.microsoft.com/en-us/azure/app-service/deploy-staging-slots) and switching to production when the staging slot is ready.  You can also use multiple instances (rather than 1) to perform a rolling update, however, the first deployment will always take a while to warm up and your site won't be accessible.

If you run into any issues you can access the SCM site: [https://[app-name].scm.azurewebsites.net/](https://[app-name].scm.azurewebsites.net/) where you can view appsettings, download container logs, etc.

Enjoy!

### Update app Docker Compose file

If you make changes to the Docker Compose file (e.g. update a container tag) then you can deploy updates like so:

```sh
az webapp config container set --resource-group votingapp --name "${APP_NAME}" --multicontainer-config-type compose --multicontainer-config-file "docker-compose-appservice.yml"
```

### Persisting the votes data

When the `db` container is deleted you'll loose your votes.  This is because container ephemeral storage is being used for the PostgreSQL database.  Also, if you scale to more than one instance, each instance will have it's own db instance and state.

To persist state and share state between instances you should mount a volume the `db` container.  Example snippet for updated `docker-compose-appservice.yaml` file:

```yaml
  db:
    image: postgres:9.4
    container_name: db
    volumes:
      - "${WEBAPP_STORAGE_HOME}/votesdata:/var/lib/postgresql/data"
```

An even better approach is to remove the `db` container altogether and store state in an Azure managed database, like the [Azure Database for PostgreSQL](https://azure.microsoft.com/en-us/services/postgresql/).  You would need to pass the DB connection string used in the `worker` and `result` apps (requires a small code change to support this).

### Clean-up

To delete all Azure resources for this demo, type the following:

```sh
az group delete --name votingapp
```

## Run the app in Docker Swarm

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:
```
docker swarm init
```
Once you have your swarm, in this directory run:
```
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder `k8s-specifications` contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services objects:
```
$ kubectl create -f k8s-specifications/
deployment "db" created
service "db" created
deployment "redis" created
service "redis" created
deployment "result" created
service "result" created
deployment "vote" created
service "vote" created
deployment "worker" created
```

The vote interface is then available on port `31000` on each host of the cluster, the result one is available on port `31001`.

## Architecture

![Architecture diagram](architecture.png)

* A Nginx proxy which handles the inbound requests for voting and result pages
* A Python webapp which lets you vote between two options
* A Redis queue which collects new votes
* A .NET worker which consumes votes and stores them in…
* A Postgres database backed by a Docker volume
* A Node.js webapp which shows the results of the voting in real time

## Note

The voting application only accepts one vote per client. It does not register votes if a vote has already been submitted from a client. Voter can change the vote at any time and result will be reflected on results page.  You can open multiple browsers or use incognito mode to register more than one vote from your computer.
