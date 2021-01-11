- [Create an Azure Kubernetes Service cluster](#create-an-azure-kubernetes-service-cluster)
- [Link with kubectl](#link-with-kubectl)
- [Deploy an application on AKS](#deploy-an-application-on-aks)
  - [Create a deployment manifest](#create-a-deployment-manifest)
  - [Apply the manifest](#apply-the-manifest)
- [Enable network access to an application](#enable-network-access-to-an-application)
  - [Create the service manifest](#create-the-service-manifest)
  - [Deploy the service](#deploy-the-service)
  - [Create an ingress manifest](#create-an-ingress-manifest)
  - [Deploy the ingress](#deploy-the-ingress)
- [Clean up resources](#clean-up-resources)
- [Reference](#reference)

# Create an Azure Kubernetes Service cluster

1. Sign in to [Azure portal](https://portal.azure.com/).
2. Create variables for the configuration values you'll reuse throughout the exercises.

    ```bash
    RESOURCE_GROUP=rg-contoso-video
    CLUSTER_NAME=aks-contoso-video
    ```

3. Run the `az group create` command to create a resource group.

    ```bash
    az group create --name $RESOURCE_GROUP --location eastus
    ```

4. Run the `az aks create` command to create an AKS cluster.

    ```bash
    az aks create \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --node-count 2 \
        --enable-addons http_application_routing \
        --enable-managed-identity \
        --generate-ssh-keys \
        --node-vm-size Standard_B2s
    ```

# Link with kubectl

1. Link your Kubernetes cluster with `kubectl` by using the following command in Cloud Shell.

   ```bash
   # This command will add an entry to your ~/.kube/config file
   az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
   ```

2. Execute the `kubectl get nodes` command to check that you can connect to your cluster and confirm its configuration.

    ```bash
    kubectl get nodes
    ```

    Output should looks like:

    ```bash
    NAME                                STATUS   ROLES   AGE    VERSION
    aks-nodepool1-14167704-vmss000000   Ready    agent   105s   v1.16.10
    aks-nodepool1-14167704-vmss000001   Ready    agent   105s   v1.16.10
    ```

# Deploy an application on AKS

## Create a deployment manifest

1. Create a manifest file for Kubernetes deployment called `deployment.yaml` by using the intergrated editor.

    ```bash
    touch deployment.yaml
    ```

2. Open the integrated editor in Cloud Shell by entering `code .`.
3. Open the `deployment.yaml` file, and add the following code section of YAML.

    ```yaml
    # deployment.yaml
    apiVersion: apps/v1 # The API resource where this workload resides
    kind: Deployment # The kind of workload we're creating
    metadata:
        name: contoso-website # This will be the name of the deployment
    ```

4. A deployment wraps a pod. A template definition is used to define the pod information within the manifest file. The template is placed in the manifest file under the deployment specification section.

   Update the `deployment.yaml` file to match the following YAML.

   ```yaml
   # deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
      name: contoso-website
   spec:
    template: # This is the template of the pod inside the deployment
        metadata: # Metadata for the pod
        labels:
            app: contoso-website
   ```

   Pods don't have given names when they're created inside deployments. The pod's name will be the deployment's name with a random ID added in the end.

   The `labels` key is used to allow deployments to find and group pods.

5. A pod wraps one or more containers. All pods have a specification section that allows user to define the containers inside the pod.

    Update the `deployment.yaml` file to match the following YAML.

    ```yaml
    # deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: contoso-website
    spec:
        template: # This is the template of the pod inside the deployment
            metadata:
            labels:
                app: contoso-website
            spec:
                containers: # Here we define all containers
            name: contoso-website
    ```

    The `containers` key is an array of container specifications because a pod can have one or more containers. The specification defines an `image`, `name`, `resources`, `ports`, and other important informaiton about the containers.

    All running pods will follow the name `contoso-website-<UUID>`, where UUID is a generated ID to identify all resources uniquely.

6. It's a good practice to define a minimum and a maximum amount of the resources the app is allowed to use from the cluster. `resources` key is used to specify this information.

    Update the `deployment.yaml` file to match the following YAML.

    ```yaml
    # deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: contoso-website
    spec:
        template: # This is the template of the pod inside the deployment
            metadata:
            labels:
                app: contoso-website
            spec:
                containers:
                    - image: mcr.microsoft.com/mslearn/samples/contoso-website
                    name: contoso-website
                    resources:
                        requests: # Minimum amount of resources requested
                            cpu: 100m
                            memory: 128Mi
                        limits: # Maximum amount of resources requested
                            cpu: 250m
                            memory: 256Mi
    ```

    Note how resources section allow user to specify the minimum resource amout as a request and the maximum resource amount as a limit.

7. The last step is to define the ports this container will expose externally through the `ports` key. The `ports` key is an array of objects, which means that a conainer in a pod can expose multiple ports with multiple names.

    Update the `deployment.yaml` file to match the following YAML.

    ```yaml
    # deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: contoso-website
    spec:
        template: # This is the template of the pod inside the deployment
            metadata:
                labels:
                    app: contoso-website
            spec:
                containers:
                    - image: mcr.microsoft.com/mslearn/samples/contoso-website
                    name: contoso-website
                    resources:
                        requests:
                            cpu: 100m
                            memory: 128Mi
                        limits:
                            cpu: 250m
                            memory: 256Mi
                    ports:
                        - containerPort: 80 # This container exposes port 80
                        name: http # We named that port "http" so we can refer to it later
    ```

    Notice how the port is named by using the `name` key. Naming ports allows user to change the exposed port without changing files that reference that port.

8. Finally, add a selector section to define the workloads the deployment will manage. The `selector` key is placed inside the deployment specification section of the manifest file. Use the `matchLabels` key to list the labels for all the pods managed by the deployment.

    Update the `deployment.yaml` file to match the following YAML.

    ```yaml
    # deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: contoso-website
    spec:
        selector: # Define the wrapping strategy
            matchLabels: # Match all pods with the defined labels
                app: contoso-website # Labels follow the `name: value` template
        template: # This is the template of the pod inside the deployment
            metadata:
                labels:
                    app: contoso-website
            spec:
                containers:
                    - image: mcr.microsoft.com/mslearn/samples/contoso-website
                    name: contoso-website
                    resources:
                        requests:
                            cpu: 100m
                            memory: 128Mi
                        limits:
                            cpu: 250m
                            memory: 256Mi
                    ports:
                        - containerPort: 80 # This container exposes port 80
                          name: http # We named that port "http" so we can refer to it later
    ```

    Save the manifest file and close the editor.

## Apply the manifest

1. In Cloud Shell, run the `kubectl apply` to submit the deployment manifest to the cluster.

   ```bash
   kubectl apply -f ./deployment.yaml
   ```

   The command should output a result similar to the following example.

   ```bash
   deployment.apps/contoso-website created
   ```

2. Run the `kubectl get deploy` command to check if the deployment was successful.

    ```bash
   kubectl get deploy contoso-website
   ```

   The command should output a result similar to the following example.

   ```bash
   NAME              READY   UP-TO-DATE   AVAILABLE   AGE
   contoso-website   0/1     1            0           16s
   ```

3. Run the `kubectl get pods` command to check if the pod is running.

   ```bash
   kubectl get pods
   ```

   The command should output a result similar to the following example.

   ```bash
   NAME                               READY   STATUS    RESTARTS   AGE
   contoso-website-7c58c5f699-r79mv   1/1     Running   0          63s
   ```

# Enable network access to an application

## Create the service manifest

Like all resources, services also have manifest files that describe how they should behave.

1. Sign in to Azure Cloud Shell.
2. In Cloud Shell, create a manifest file for AKS called `service.yaml`.

    ```yaml
    touch service.yaml
    ```

3. Open the integrated editor in Cloud Shell by entering `code .`.
4. Open the `service.yaml` file, and add the following code section of YAML.

    ```yaml
    #service.yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: contoso-website
    ```

    In this code, first two keys are added to tell Kubernetes the `apiVersion` and `kind` of manifest which has been created. The `name` is the name of the service. It'll be used to identify and query the service information when `kubectl` is used.

5. The behavior of service needs to be defined in the specification section of the manifest file. The first behavior needs to be added is the type of service. Set the `type` key to `clusterIP`.

    ```yml
    #service.yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: contoso-website
    spec:
        type: ClusterIP
    ```

6. `selector` section in the manifest file is used to define the pods the service will group and provide coverage. Add the `selector`, and set the `app` key value to the `contoso-website` label of the pods as specificied in the earlier deployment's manifest file.

    Update the service.yaml file to match the following YAML.

    ```yml
    #service.yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: contoso-website
    spec:
        type: ClusterIP
        selector:
        app: contoso-website
    ```

7. `ports` section in manifest file is used to define the port-forwarding rules. The service must accept all TCP requests on port 80 and forward the request to the HTTP target port for all pods matching the selector value defined earlier.

    Update the service.yaml file to match the following YAML.

    ```yml
    #service.yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: contoso-website
    spec:
        type: ClusterIP
        selector:
            app: contoso-website
        ports:
            - port: 80 # SERVICE exposed port
              name: http # SERVICE port name
              protocol: TCP # The protocol the SERVICE will listen to
              targetPort: http # Port to forward to in the POD
    ```

8. Save the manifest file, and close the editor.

## Deploy the service

1. In Cloud Shell, run the `kubectl apply` command to submit the service manifest to your cluster.

    ```bash
    kubectl apply -f ./service.yaml
    ```

    The command should output a result similar to the following example:

    ```bash
    service/contoso-website created
    ```

2. Run the `kubectl get service` command to check if the deployment was successful.

    ```bash
    kubectl get service contoso-website
    ```

    The command should output a result similar to the following example. Make sure the column `CLUSTER-IP` is filled with an IP address and the column `EXTERNAL-IP` is `<none>`. Also, make sure the column `PORT(S)` is defined to `80/TCP`.

    ```bash
    NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    contoso-website   ClusterIP   10.0.158.189   <none>        80/TCP    42s
    ```

    With the external IP set to `<none>`, the application isn't available to external clients. The service is only accessible to the internal cluster.

## Create an ingress manifest

To expose your website to the world via DNS, you must create an ingress controller.

1. In Cloud Shell, create a manifest file for the Kubernetes service called `ingress.yaml`.
2. Open the integrated editor in Cloud Shell by entering `code .`
3. Open the `ingress.yaml` file, and add the following code section of YAML.

   ```yml
   #ingress.yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
       name: contoso-website
   ```

   In this code, first two keys are added to tell Kubernetes the `apiVersion` and `kind` of manifest that has been created. The `name` is the name of the ingress. It;ll be used to identify and query the ingress information when `kubectl` is used.

4. Create an `annotations` key inside the `metadata` section of the manifest file called to use the HTTP application routing add-on for this ingress. Set the key to `kubernetes.io/ingress.class` and a value of `addon-http-application-routing`.

   Update the `ingress.yaml` file to match the following YAML.

    ```yml
    #ingress.yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
        name: contoso-website
        annotations:
            kubernetes.io/ingress.class: addon-http-application-routing
    ```

5. Set the fully qualified domain name (FQDN) of the host allowed access to the cluster.

    In Cloud Shell, run the `az network dns zone` list command to query the Azure DNS zone list.

    ```bash
    az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME -o tsv --query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName
    ```

6. Copy the output, and update the `ingress.yaml` file to match the following YAML. Replace the `<zone-name>` placeholder value with the `ZoneName` value you copied.

    Update the `ingress.yaml` file to match the following YAML.

    ```yaml
    #ingress.yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
        name: contoso-website
        annotations:
            kubernetes.io/ingress.class: addon-http-application-routing
    spec:
        rules:
            - host: contoso.<zone-name> # Which host is allowed to enter the cluster
    ```

7. Next up, add the back-end configuration to ingress rule. Create a key named `http` and allow the `http` protocol to pass through. Then, define the `paths` key that will allow you to filter whether this rule applies to all paths of the website or only some of them.

    Update the `ingress.yaml` file to match the following YAML.

    ```yaml
    #ingress.yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
        name: contoso-website
        annotations:
            kubernetes.io/ingress.class: addon-http-application-routing
    spec:
        rules:
            - host: contoso.<uuid>.<region>.aksapp.io
              http:
                paths:
                - backend: # How the ingress will handle the requests
                    serviceName: contoso-website # Which service the request will be forwarded to
                    servicePort: http # Which port in that service
                  path: / # Which path is this rule referring to

    ```

8. Save the manifest file, and close the editor.

## Deploy the ingress

# Clean up resources

# Reference

- [Deploy a containerized application on Azure Kubernetes Service](https://docs.microsoft.com/en-us/learn/modules/aks-deploy-container-app/)

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
