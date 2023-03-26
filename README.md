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

![](https://miro.medium.com/v2/resize:fit:1140/format:webp/1*AWl1dprZ9UvUTXtg3JtbPg.png)
In this case we are using it to authenticate to our azure container registry and push container images/helm charts to it. The required permissions have been granted during the terraform deployment. There are a couple of advantages to using scripts rather than the built in tasks — the results are reproduce-able, the process is easier to version, we can copy it over to another repo and as a bonus we can run the scripts offline.
## PIPELINE:
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uaI8izGiqK7FO8faxfE4iQ.png)



The first one is the build stage (also named Build) which should not produce any artefacts — since everything will be already versioned inside our azure container registry. The only exception are the build scripts which need to match the git changeset of our release process that we are currently in- which is why we are publishing the scripts folder as part of our build process to the staging directory of each release process instance.

![](https://miro.medium.com/v2/resize:fit:1050/format:webp/1*bI8RvzOJQqKAuGHrZgAGcg.png)

As for the deployment stages we are defining a dependancy of the first deployment environment (here called DevDeploy) to the Build stage, give it a random name dev1 (the exact name does not matter but needs to be the same if we have multiple microservices in different release pipelines deploying to the same environment) and introduce the Azure KeyVault instance name dzphoenix-180-vault as a stage variable. All the variables will be accessible under the same name in our bash scripts.

The script deploy_multicalulator.sh will use the azure cli context to authenticate to the configured Azure KeyVault instance, collect the environment specific variables and perform a deployment to the corresponding AKS cluster.
![](https://miro.medium.com/v2/resize:fit:1218/format:webp/1*wrWsYNQaoBs0FKvDRAecIg.png)

Since we also want to make sure that our deployment actually works we are triggering a traffic script that will in our case do nothing else but check if the ingress controller actually serves our application.
![](https://miro.medium.com/v2/resize:fit:1246/format:webp/1*omZZ8ozujt05E_KxrmVlCg.png)

Just in case that something goes wrong we are triggering a rollback script that will perform a helm rollback to the latest working helm deployment.


Just in case that something goes wrong we are triggering a rollback script that will perform a helm rollback to the latest working helm deployment.
!{](https://miro.medium.com/v2/resize:fit:1250/format:webp/1*OKr6LV9_skVKWDotP-kTpA.png)

Assuming everything works fine we will see our change getting automatically deployed across all configured stages.



![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*zEgcBT_M8mlxHgHiTpy67w.png)
If you check the ‘Environments’ tab on the left you can see every release status across all deployments — which is very useful if you have not only one release pipeline, but a dedicated pipeline for each microservice.

You can retrieve the public dns (which is depending on the public ip of the ingress controller inside the cluster) by looking at the log output of the routetraffic task.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*PAhODJIuVwKibryPXfQNLQ.png)
Assuming your deployment pipeline works you can now go ahead change sourcecode and see your changes flowing through all of your environments without interrupting your users.



![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uwnXCfbpzOGp0shrQYpKUw.png)

