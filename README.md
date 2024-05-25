# End to end Text-Summarizer

## Workflows

1. Update config.yaml
2. Update params.yaml
3. Update entity
4. Update the configuration manager in src config
5. update the components
6. update the pipeline
7. update the main.py
8. update the app.py


# How to run?
### STEPS:

Clone the repository

```bash
git clone https://github.com/Bouallegui-Moudhaffer/Text-Summarizer.git
cd Text-Summarizer
```
### STEP 01- Create a conda environment after opening the repository

```bash
conda create -n summary python=3.9 -y
```

```bash
conda activate summary
```


### STEP 02- install the requirements
```bash
pip install -r requirements.txt
```


```bash
# Finally run the following command
python app.py
```

Now,
```bash
Now, open your browser and navigate to your local host and port to access the application.
```


```bash
Author: Moudhaffer Bouallegui
Data Scientist
Email: boualleguimoudhaffer@gmail.com
Website: https://www.moudhafferbouallegui.me
LinkedIn: https://www.linkedin.com/in/moudhafferbouallegui
```

# Docker Installation and Setup

### Step 1: Install Docker

#### On Ubuntu:
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install -y docker-ce

#### On Windows or macOS:
    Follow the instructions on the Docker website to install Docker Desktop.

### Step 2: Build and Run Docker Container Locally

#### Build the Docker image:
    docker build -t text-summarizer .

#### Run the Docker container:
    docker run -d -p 8080:8080 text-summarizer

### Navigate to http://localhost:8080 to access the application.

# Deploy to Azure

### Step 1: Create Azure Resources

#### Create a Resource Group
    az group create --name {myResourceGroup} --location eastus

#### Create Azure Container Registry (ACR)
    az acr create --resource-group {myResourceGroup} --name {myDockerImage} --sku Basic

#### Create an Azure VM
    az vm create \
      --resource-group {myResourceGroupe} \
      --name {myVirtualMachine} \
      --image UbuntuLTS \
      --size Standard_D4s_v3 \
      --admin-username {azureuser} \
      --generate-ssh-keys

### Step 2: Install Docker on Azure VM

#### SSH into your VM:

    ssh {azureuser}@<your-vm-ip>

#### Install Docker:

    sudo apt-get update
    sudo apt-get install -y docker.io
    sudo usermod -aG docker {azureuser}
    newgrp docker

### Step 3: Push Docker Image to Azure Container Registry (ACR)

#### Login to ACR:

    az acr login --name {myDockerImage}

#### Tag the Docker image:

    docker tag text-summarizer {myDockerImage}.azurecr.io/text-summarizer:latest

#### Push the Docker image:

    docker push {myDockerImage}.azurecr.io/text-summarizer:latest

### Step 4: Deploy Docker Container on Azure VM

#### SSH into your VM (if not already done):

    ssh {azureuser}@<your-vm-ip>

#### Login to ACR from the VM:

    az acr login --name {myDockerImage}

#### Pull and run the Docker image:

    docker pull {myDockerImage}.azurecr.io/text-summarizer:latest
    docker run -d -p 80:8080 {myDockerImage}.azurecr.io/text-summarizer:latest

### Navigate to http://<your-vm-ip> to access the application.


# Azure CI/CD Deployment with GitHub Actions

## 1. Login to Azure Portal

## 2. Create a Service Principal for Deployment

	az ad sp create-for-rbac --name "myApp" --role contributor --scopes /subscriptions/{subscription-id}/resourceGroups/{myResourceGroup} --sdk-auth

Save the output JSON, which contains the credentials needed for GitHub Secrets.

### Step 3: Set Up GitHub Secrets

#### Add the following secrets in your GitHub repository under Settings > Secrets:

    AZURE_CREDENTIALS: Paste the JSON output from the service principal creation.
    AZURE_CONTAINER_REGISTRY: The name of your ACR (e.g., tsdockerimage).
    AZURE_RESOURCE_GROUP: The name of your resource group (e.g., textSummarizationRG).
    AZURE_VM_NAME: The name of your VM (e.g., tsVM).
    AZURE_VM_IP: The public IP address of your VM.
    AZURE_SSH_PRIVATE_KEY: Your SSH private key content.

### Step 4: Create GitHub Actions Workflow

#### Create a .github/workflows/deploy.yml file in your repository with the following content:

    name: workflow

    on:
      push:
        branches:
          - main
        paths-ignore:
          - 'README.md'
    
    permissions:
      id-token: write
      contents: read
    
    jobs:
      integration:
        name: Continuous Integration
        runs-on: ubuntu-latest
        steps:
          - name: Checkout Code
            uses: actions/checkout@v3
    
          - name: Lint code
            run: echo "Linting repository"
    
          - name: Run unit tests
            run: echo "Running unit tests"
    
      build-and-push-acr-image:
        name: Continuous Delivery
        needs: integration
        runs-on: ubuntu-latest
        steps:
          - name: Checkout Code
            uses: actions/checkout@v3
    
          - name: Install Utilities
            run: |
              sudo apt-get update
              sudo apt-get install -y jq unzip
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          - name: Login to Azure Container Registry
            run: |
              az acr login --name ${{ secrets.AZURE_CONTAINER_REGISTRY }}
    
          - name: Build, tag, and push image to Azure Container Registry
            env:
              ACR_REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io
              IMAGE_TAG: latest
            run: |
              docker build -t $ACR_REGISTRY/text-summarizer:$IMAGE_TAG .
              docker push $ACR_REGISTRY/text-summarizer:$IMAGE_TAG
    
      Continuous-Deployment:
        needs: build-and-push-acr-image
        runs-on: ubuntu-latest
        steps:
          - name: Checkout
            uses: actions/checkout@v3
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          - name: SSH into Azure VM and deploy
            uses: appleboy/ssh-action@master
            with:
              host: ${{ secrets.AZURE_VM_IP }}
              username: {azureuser}
              key: ${{ secrets.AZURE_SSH_PRIVATE_KEY }}
              script: |
                az acr login --name ${{ secrets.AZURE_CONTAINER_REGISTRY }}
                docker pull ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io/text-summarizer:latest
                docker stop $(docker ps -q --filter ancestor=${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io/text-summarizer:latest) || true
                docker rm $(docker ps -q --filter ancestor=${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io/text-summarizer:latest) || true
                docker run -d -p 80:8080 --name=text-summarizer ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io/text-summarizer:latest
                docker system prune -f

### Step 5: Commit and Push

#### Commit the .github/workflows/deploy.yml file to your repository and push it to the main branch.

    git add .github/workflows/deploy.yml
    git commit -m "Add GitHub Actions workflow for CI/CD"
    git push origin main

### Step 6: Monitor and Verify

#### Monitor GitHub Actions:

    Go to the Actions tab in your GitHub repository to monitor the workflow runs.
#### Verify Deployment:
    
SSH into your Azure VM and check the Docker container logs to ensure the deployment was successful.
    
    ssh {azureuser}@<your-vm-ip>
    docker logs text-summarizer

Open your browser and navigate to your VM's public IP address to access the application.


### By following these updated instructions, you can clone the repository, set up the environment, install and use Docker locally, deploy the application to Azure, and automate the deployment using GitHub Actions.
### If you encounter any issues or need further assistance, feel free to reach out!

## Author
### Moudhaffer Bouallegui
### Data Scientist
### Email: boualleguimoudhaffer@gmail.com
### Website: https://www.moudhafferbouallegui.me
### LinkedIn: https://www.linkedin.com/in/moudhafferbouallegui
