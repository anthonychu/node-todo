# Deploy a containerized web app to Azure App Service

## Introduction

### Technologies used

* Docker
* Azure Container Registry
* Azure App Service (Web App for Containers)

### Prerequisites

The following software is required to build this tutorial.

* Git
* Node.js (Version 8 or higher)
* Docker Desktop
* Azure CLI
* VS Code (recommended)
    - Azure Tools extension


## Run the application locally


### Clone the repository

The application you'll be deploying is a simple Node.js application with a MongoDB database.

1. Create a new directory and clone the repository that contains the Node.js application.

    ```
    git clone https://github.com/anthonychu/node-todo.git
    ```

1. In VS Code, open the folder containing the cloned repository.

### Build and run the application

The application requires access to a MongoDB server when it is run. You'll use Docker Compose to:

* Build and containerize the Node.js application
* Start a MongoDB server in a container
* Start the application in a container, which connects to MongoDB

#### Review the Docker and Docker Compose files

1. In VS Code, open and review the contents of the **Dockerfile**. It specifies the base image (`node:8-alpine`) and the steps used to build your container image.

1. Open and review the contents of the **docker-compose.yml** file. It instructs docker to start two containers, one with a MongoDB server and one with the main application.

#### Build and start the application with Docker Compose

1. In the directory containing **docker-compose.yml**, run the following command that will build and run the application:

    ```
    docker-compose up -d
    ```

1. Open a browser and navigate to the application at `http://localhost:8080`.


## Host the container image in Azure Container Registry

Before you can deploy your application in Azure, you need to make the application's Docker image available in a container registry. You'll use Azure Container Registry to build and host your container image for this lab.


### Create an Azure Container Registry

1. Before you can use the Azure CLI, make sure you have logged in. Run the following command and follow the instructions to log in.

    ```
    az login
    ```

1. View your subscriptions in table format:

    ```
    az account list -o table
    ```

1. Confirm that the subscription you would like to use for this lab is listed as the **default**. To change the default subscription, run this command:

    ```
    az account set -s "Name or SubscriptionId"
    ```

1. A resource group is a logical container for resources in Azure. Create a resource group named **node-todo-lab**:

    ```
    az group create -n node-todo-lab -l westus2
    ```

1. Create an Azure Container Registry (ACR) using a unique name.

    ```
    az acr create -n <registryname> -g node-todo-lab -l westus2 --sku Standard --admin-enabled true
    ```

    > Note: The admin account is required for Azure App Service to authenticate with ACR.

### Build and push the Docker image to ACR

1. Build the Docker image using ACR. This command tags the image as `nodetodo:latest` and pushes it to your ACR.

    ```
    az acr build -t nodetodo:latest -r <registryname> .
    ```

1. Confirm that `nodetodo:latest` exists in your Azure Container Registry:

    ```
    az acr repository show-tags -n <registryname> --repository nodetodo
    ```


## Deploy the application to Azure

### Create an Azure Cosmos DB account

1. In the same resource group, create a Cosmos DB account (MongoDB API) with a unique name.

    ```
    az cosmosdb create -n <cosmosdbname> -g node-todo-lab --locations "westus2=0" --kind MongoDB
    ```

    > Note: Cosmos DB supports replication to multiple regions. Multiple regions and their failover priorities can be passed to the `--locations` parameter (failover priority starts at 0 which indicates the primary location).

### Deploy the application to Azure App Service

1. Retrieve information about your container registry and image into variables.

    ```
    ACR_SERVER=$(az acr show -n <registryname> --query loginServer -o tsv)
    ACR_USERNAME=$(az acr credential show -n <registryname> --query username -o tsv)
    ACR_PASSWORD=$(az acr credential show -n <registryname> --query passwords[0].value -o tsv)
    IMAGE_NAME=$ACR_SERVER/nodetodo:latest
    ```

    > Note: These commands are in bash format. They may differ slightly in other shells.

1. Create an Azure App Service plan with a unique name. An App Service plan can host one or more applications.

    ```
    az appservice plan create -n <planname> -g node-todo-lab -l westus2 --sku S1  --is-linux
    ```

1. Create a web app with a unique name.

    ```
    az webapp create -n <webappname> -g node-todo-lab -p <planname> --deployment-container-image-name $IMAGE_NAME
    ```

1. Configure the web app with the credentials to retrieve the image from ACR.

    ```
    az webapp config container set -n <webappname> -g node-todo-lab -i $IMAGE_NAME -u $ACR_USERNAME -p $ACR_PASSWORD
    ```

1. Set an environment variable named **MONGODB_URL** with the connection string to the Cosmos DB account you created earlier.

    ```
    CONNECTION_STRING=$(az cosmosdb list-connection-strings -n <webappname> -g node-todo-lab --query connectionStrings[0].connectionString -o tsv)
    az webapp config appsettings set -n <webappname> -g node-todo-lab --settings MONGODB_URL=$CONNECTION_STRING
    ```

1. Enable container logging.

    ```
    az webapp log config -n <webappname> -g node-todo-lab --docker-container-logging filesystem
    ```

1. Retrieve the URL for your web app. Open it in a browser.

    ```
    echo https://$(az webapp show -n <webappname> -g node-todo-lab --query defaultHostName -o tsv)/
    ```

1. Stream the container logs in real-time.

    ```
    az webapp log tail -n <webappname> -g node-todo-lab
    ```


## Summary

Congratulations! You have successfully containerized and deployed an application to Azure App Service and Azure Cosmos DB!

