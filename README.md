Continuous building, gating and releasing using AzureDevOps, Helm3, Application Insights and AKS with automated rollback

## Architecture 

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*kZnobVLbOzuN-isKYLCKNA.png)


To make it more realistic for an enterprise setup here are a couple of design considerations that I consider worth pursuing:

Your process implementation and all assets should be versioned and stored with your source code — meaning you can re-use the process for another microservice, maybe run it from another repo and it will still work.
You should not bind your process to the particular implementation of your CI/CD toolchain — meaning we will be using scripts for implementing our build and release steps and not Azure DevOps tasks because they require manual configuration and are hard to test (without Azure DevOps).
All environment specific variables should be maintained in a secure store and injected into your pipelines — meaning all variables will be sourced from an azure KeyVault instance, which will be dedicated to an environment.


After running the deployment scripts please remember the output with the azure devops credentials, because we will need them later.


* Run the terraform scripts
* You should end up with a dedicated resource group for your azure container registry, one resource group for each environment which contains the 
- Azure KeyVault (which should also already contain the configuration variables for that environment)
- Application Insights
- Azure Redis Cache
- VNETS
- Azure Monitor 
- AKS Cluster with an already deployed Nginx Ingress Controller inside plus one extra resource group for your Kubernetes worker nodes. 
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*g2FKAODE_U-Lejhu9sVhGg.png)

Assuming you have the right permissions you will also have two service principals — one used by your AKS cluster and one is for your Azure Devops service connection to authenticate from your pipeline runners to your azure resources and especially your Azure Container Registry and Azure KeyVault.


Next you should create your Azure DevOps instance by going to dev.azure.com to create your project within your organisation. For the moment I would recommend you to not import the source code into your azure devops repo but rather leave the code on your github fork. Instead lets go directly to Pipelines and import the existing pipeline from your github repo located here: /.azuredevops/calculator_build_deploy.yaml into azure devops.


Go via Project settings -> Pipelines -> Service Connections and create a new Azure Resource Manager connection with Service Principal (manual) and Enter all the values from the terraform deployment output. Also make sure that your Service Connection Name is either set to defaultAzure or matches the name in the pipeline template.

## PIPELINE:
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uaI8izGiqK7FO8faxfE4iQ.png)
