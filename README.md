Continuous building, gating and releasing using AzureDevOps, Helm3, Application Insights and AKS with automated rollback

## Architecture 

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/0c49923f-7294-4757-b30a-f954a8157649)


To make it more realistic for an enterprise setup here are a couple of design considerations that I consider worth pursuing:

1. Your process implementation and all assets should be versioned and stored with your source code — meaning you can re-use the process for another microservice, maybe run it from another repo and it will still work.
2. You should not bind your process to the particular implementation of your CI/CD toolchain — meaning we will be using scripts for implementing our build and release steps and not Azure DevOps tasks because they require manual configuration and are hard to test (without Azure DevOps).
3. All environment specific variables should be maintained in a secure store and injected into your pipelines — meaning all variables will be sourced from an azure KeyVault instance, which will be dedicated to an environment.




After running the deployment scripts please remember the output with the azure devops credentials, because we will need them later.


* Run the terraform scripts
* You should end up with a dedicated resource group for your azure container registry, one resource group for each environment which contains the 
- Azure KeyVault (which should also already contain the configuration variables for that environment)
- Application Insights
- Azure Monitor 
- AKS Cluster with an already deployed Nginx Ingress Controller inside plus one extra resource group for your Kubernetes worker nodes. 
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*g2FKAODE_U-Lejhu9sVhGg.png)

Assuming you have the right permissions you will also have two service principals — one used by your AKS cluster and one is for your Azure Devops service connection to authenticate from your pipeline runners to your azure resources and especially your Azure Container Registry and Azure KeyVault.


Next you should import the existing pipeline located here: /.azuredevops/calculator_build_deploy.yaml into azure devops pipeline section .


Go via Project settings -> Pipelines -> Service Connections and create a new Azure Resource Manager connection with Service Principal (manual) and Enter all the values from the terraform deployment output. Also make sure that your Service Connection Name is either set to defaultAzure or matches the name in the pipeline template.

![](https://miro.medium.com/v2/resize:fit:1140/format:webp/1*AWl1dprZ9UvUTXtg3JtbPg.png)

In this case we are using it to authenticate to our azure container registry and push container images/helm charts to it. The required permissions have been granted during the terraform deployment. There are a couple of advantages to using scripts rather than the built in tasks — the results are reproduce-able, the process is easier to version, we can copy it over to another repo and as a bonus we can run the scripts offline.



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


 Application map of your Application Insight resource for each environment
 
 ![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/b6583b45-bb51-43cd-a424-519a6fbba169)

# ISSUES IDENTIFIED 

-> The deployment process performs a rollback in even of a deployment failure, but it does not prevent us from suffering application downtime while the broken deployment will be rolled back.

# Blue - Green Deployment 

All you have to do is configure the AZURE_CONTAINER_REGISTRY_NAME for your deployment and the AZURE_KEYVAULT_NAME for each environment secret store in the pipeline after you have imported it into your azure devops project.

single azure load balancer in front of our Nginx ingress controller in the AKS cluster

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/a9457545-d7f6-4473-8b32-fc55f6e75766)


To make the automation work, we also need an extra helm variable called “canary” which will be used to define which of the two deployments is currently the canary. Before deploying a new version we will read back all deployed helm charts from the cluster, determine if there is already a canary deployment or not and perform a new deployment in either the blue or the green slot — which ever is currently not used or in a older version than the other slot.

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/4cb4fed5-e8b1-40bb-b6fa-02dc333fa5ca)


In the new canary deployment we will feed in the canary=true helm variable, which, which will ensure the configuration for the annotation value for the Nginx canary header in the ingress object. At the same time we have defined the weight of the canary to be 0, which will ensure that no traffic will be routed there unless the header is set.

As you can see below in the initial version we only have the blue version 3.0.378 deployed in the blue-calculator namespace.

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/4da6686a-6658-4168-9977-ddcd41e26de9)


Now in step 5 of our process we are deploying the green version 3.0.379 into the green-calculator namespace — but with the canary annotation in the ingress object.

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/bc5e6895-3558-470c-8122-c70c4fcd3c4e)



By setting the canary deployment variable we have ensured that normal production traffic will still be routed to the blue version. Only if we are setting the canary header “canary: always” in the HTTP request we are ending up in the green version.



![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/0fc2b3f1-dc87-403d-8a77-ffdba1f421a5)

Now we will simply delete the existing production slot and as a last step do one more upgrade of the existing green slot with canary=false to promote it to be the new production slot and ensure we can start over at the beginning when the next release with a new canary gets deployed.

![image](https://github.com/shobitgg/AZURE-DEVSECOPS/assets/41900814/fcf6dcf6-0644-4de2-b705-3b4cc5667c8d)

Since we are using Nginx and Application Insights in our app that means we are getting very detailed metrics, logs and dashboards which will allow us to compare the performance and functionality of our new apps. With a little tuning we can easily set up a Grafana dashboard that allows us to compare both deployments side by side.

Thanks you for going through this long post !

if you need to connect : connect on Linkedin 


