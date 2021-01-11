- [Create a container](#create-a-container)
- [Reference](#reference)

# Create a container

1. Sign in to [Azure portal](https://portal.azure.com/).
2. Open Azure Cloud Shell from Azure portal.
3. Create a new resource group with the name **learn-deploy-aci-rg**.

    ```bash
    # location can be canged
    # learn-deploy-aci-rg can be a name, a Docer image, and an Azure resource group
    az group create --name learn-deploy-aci-rg --location eastus
    ```

    A container is created by providing a name, a Docer image, and an Azure resource group to the `az container create` command.

    Optionally, container can be exposed to the Internet by specifying a DNS name label.

4. Provide a DNS name, **must be unique one**, to expose the container to the Internet.

    ```bash
    # for learning purpose
    DNS_NAME_LABEL=aci-demo-$RANDOM
    ```

5. Run the following `az container create` command to start a container instance.

   ```bash
   # $DNS_NAME_LABEL specifics the DNS name
   # the image name, microsoft/aci-helloworld, refers to a Docker image hosted on Docker Hub that runs a basic Node.js web application.
   az container create \
        --resource-group learn-deploy-aci-rg \
        --name mycontainer \
        --image microsoft/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL \
        --location eastus
   ```

6. When the `az container create` command completes, run `az container show` to check its status.

    ```bash
    az container show \
        --resource-group learn-deploy-aci-rg \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table
    ```

7. From a browser, navigate to your container's FQDN to see it running.

# Reference

[Run Docker containers with Azure Container Instances](https://docs.microsoft.com/en-us/learn/modules/run-docker-with-azure-container-instances/)
