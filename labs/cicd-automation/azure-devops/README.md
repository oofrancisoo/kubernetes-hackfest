# Lab: Azure DevOps CI/CD

This workshop will guide you through building Continuous Integration (CI) and Continuous Deployment (CD) pipelines with Azure DevOps for use with Azure Kubernetes Service. The pipeline will utilize Azure Container Registry to build the images and Helm for application updating. 

## Prerequisites 

* Complete previous labs:
    * [Azure Kubernetes Service](../../create-aks-cluster/README.md)
    * [Build Application Components in Azure Container Registry](../../build-application/README.md)
    * [Helm Setup and Deploy Application](../../helm-setup-deploy/README.md)

## Instructions

The general workflow/result will be as follows:

- Push code to source control
- Trigger a continuous integration (CI) build pipeline when project code is updated via Git
- Package app code into a container image (Docker Image) created and stored with Azure Container Registry
- Trigger a continuous deployment (CD) release pipeline upon a successful build
- Deploy container image to AKS upon successful a release (via Helm chart)
- Rinse and repeat upon each code update via Git
- Profit

![](workflow.png)


#### Setup Azure DevOps Project

1. Create a Azure DevOps organization/account. Follow the steps here: https://docs.microsoft.com/en-us/azure/devops/user-guide/sign-up-invite-teammates?view=vsts

2. Create New Project in Azure DevOps

    * Name your project "azure-devops-aks" and give it a description.
    * Leave the Version control as Git

    ![](azure-do-new-project-new.png)

3. Click on "Repos" and "Import" and use the source from this repo

   https://github.com/Azure/kubernetes-hackfest
   
    ![](azure-do-import.png)

#### Create Build Pipeline

1. Create an empty build pipeline. Click on "Pipelines" then "New pipeline"
2. Click on "Use Visual designer"
3. In "Select a source," use `Azure Repos Git` and ensure it is pointing to your newly built repo (this is the default), Click Continue
    > Note that we are using the master branch here. Normally we would use other branches and PR's. For simplicity, we are using master just for this lab.

4. Select to "Empty job" at the top of the page
5. Leave the name as "azure-devops-aks-CI"
6. Under Agent pool: Change the Agent to use the "Hosted Ubuntu 1604"
7. Click the plus sign by Agent job 1 to add a task
8. Search tasks for "Azure CLI" and add the Azure CLI task

    ![](azure-do-azurecli.png)

9. Click on the Azure CLI task and choose your Azure subscription and click `Authorize`
10. Choose "Inline script" and enter the following (be sure to replace the ACR name with yours). Notice how we create a dynamic image tag using our build ID from VSTS.

    > **Note:** Make sure to add your Azure Container Registry name to the first line of the script below.

    ```
    export ACRNAME='<replace>'
    export IMAGETAG=vsts-$(Build.BuildId)

    az acr build -t hackfest/data-api:$IMAGETAG -r $ACRNAME --no-logs ./app/data-api
    az acr build -t hackfest/flights-api:$IMAGETAG -r $ACRNAME --no-logs ./app/flights-api
    az acr build -t hackfest/quakes-api:$IMAGETAG -r $ACRNAME --no-logs ./app/quakes-api
    az acr build -t hackfest/weather-api:$IMAGETAG -r $ACRNAME --no-logs ./app/weather-api
    az acr build -t hackfest/service-tracker-ui:$IMAGETAG -r $ACRNAME --no-logs ./app/service-tracker-ui  
    ```

11. Add another task and search for "Publish Build Artifacts". Click the task and use "charts" for the artifact name and browse to the charts folder for the "Path to publish"

    ![](azure-do-artifact.png)

11. Click on 'Triggers' and toggle 'Enable continuous integration'
12. Test this by clicking "Save & queue" and providing a comment
13. Click on "Builds" to check result


#### Create Deployment Pipeline

In the deployment pipeline, we will create a Helm task to update our application. 

    > Note: To save time, we will only deploy the service-tracker-ui application in this lab. 

1. Under Pipelines click on "Releases"
2. Click the "New pipeline" button
3. Select to "Empty job"
4. Name the pipeline "AKS Helm Deploy" (it will default to "New release pipeline")
5. Name the stage "dev" and click the 'x' to close the window
6. Click on "+ Add" next to Artifacts
7. In "Source (build pipeline)", select the build we created earlier (should be named "azure-devops-aks-CI")

    ![](azure-do-release-artifact.png)

8. Click on the "Add" button 
9. Click on the lightning bolt next to the Artifact we just created and enable "Continuous deployment trigger"
10. Click on "1 job, 0 task" button to view stage tasks
11. Click on "Agent job" and change the agent pool to "Hosted Hosted Ubuntu 1604" in the drop down
12. On the Agent job, click the "+" to add a Task
13. Search for "helm" and add the task called "Package and deploy Helm charts"
14. Click on the task (named "helm ls") to configure all of the settings for the release
    
    * Select your Azure subscription in the dropdown and click "Authorize"
    * Select the Resource Group and AKS Cluster
    * For the Command select "upgrade"
    * For Chart type select "File Path"
    * For Chart path, click the "..." button and browse to the "service-trakcer-ui" chart in the charts directory
    * For the Release Name, enter `service-tracker-ui`
    * For Set Values you will need to fix the ACR server to match your ACR server name and the imageTag needs to be set. 
        Eg - `deploy.acrServer=acrhackfestbrian123.azurecr.io,deploy.imageTag=vsts-$(Build.BuildId)`
    * For testing, you can add `--timeout 60` to the Arguments field. The default value is 300 seconds, which means your pipeline can take 5 minutes to fail if there is a problem.

    ![](azure-do-helm-task.png)

#### Run a test build

1. In Azure DevOps, click on Builds and click the "Queue" button
2. Monitor the builds and wait for the build to complete

    ![](azure-do-build.png)

3. The release will automatically start when the build is complete (be patient, this can take some time). Review the results as it is complete. 

    ![](azure-do-release.png)

4. Now kick-off the full CI/CD pipeline by making an edit to the service-tracker-ui frontend code in the Azure DevOps code repo.

#### Next Lab: [Networking](../../networking/README.md)

## Troubleshooting / Debugging

## Docs / References

* Blog post by Jessica Dean. http://jessicadeen.com/tech/microsoft/how-to-deploy-to-kubernetes-using-helm-and-vsts 
