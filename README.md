## Deployment of Sapio Analytics Server (Single Instance)

1. Modify the docker-compose.yaml to ensure your port outbound was desirable.
2. Modify the docker-compose.yaml to use a custom API key.
2. Double check docker-compose.yaml to make sure the image path is the version you wanted. (You should have used a tagged build)
3. Use the command "docker-compose pull" to grab latest image. (Unless this is Sapio dev building from scratch.)
4. Use the command "docker-compose up" to create instance.
   Note you can use --detached so the lifecycle does not end when you exit console.
   You can also use -p to specify a custom project name but usually more convenient to rename the parent directory.
5. The best way to specify a public/private certificate is to generate a PKCS12 file with both private key and certificate inside, encrypted with a password.
   Then, use the base64 command to get the base64 string and push it under SAPIO_EXEC_SERVER_KEYSTORE_BASE64 ENV in the docker-compose file.
   If the password is different, change the environment variable "SAPIO_EXEC_SERVER_KEYSTORE_PASSWORD" in the docker-compose file.
   The old way of replacing keystore file can still be used if these ENV variables are not present in the container.

The app or the Sapio Platform (BLS) server may need reboot to take effect.

## Use Docker Images
All Sapio analytic server docker images are publicly available [here](https://hub.docker.com/repository/docker/sapiosciences/sapio_analytics_server/).
The latest image is tagged as "latest" and the versioned images are tagged as "X.Y".
**Do not use the latest image, unless you know what you are doing. Instead, use the tagged image that corresponds to your deployment's Sapio platform version.**

## Sapio App Setup
After deploying the Sapio Analytics Serer and configured the ClientSettings.properties, you will need to redirect the
binary locations in Sapio Analytics Settings to the correct locations installed in this container.

The default value of the baseline synergy are the correct values.
To verify, go to App Setup => Configuration Manager.
Navigate to "Analytics" menu
The values should be as follows.
### Native Analytics Form
1. python3
2. R
3. /opt/sapiosciences/rtranslator/translate.sh
### CRISPR Form
1. /data/indexes
2. GRCh38_latest_genomic
   If any values are incorrect, please make adjustment as necessary and save.
   The changes will take effect immediately.


### Connection Properties
You need to have Environmental Variables set up on the shell that launches the Sapio BLS (Sapio Platform Server).
The following environment variables are required.
1. *SapioNativeExecAPIKey*=The exact API key string you have set up as a random string in docker-compose file earlier. (YOU SHOULD HAVE MODIFIED THIS.)
2. *SapioNativeExecHost*=Where the analytic server is located w.r.t. ethernet interface from the Sapio BLS. It can be hostname or IP.
3. *SapioNativeExecPort*=The port in docker-compose file listening in the analytic server. Check analytic server inbound firewall (or its gateway's if cluster) to allow Sapio BLS connection.
3. *SapioNativeExecTrustStoreData*=The base64 string of the PKCS12 file you have set up in docker-compose file earlier. (YOU SHOULD HAVE MODIFIED THIS.)

You need the ENV to be *ACTIVE* and *EXPORTING TO NEW CHILD PROCESSES* under the shell that launches Sapio BLS.
The "cheap way" to do this is through a shell scripting invoking a static .env text file, before executing the line to launch Java for Sapio BLS.
```shell
export $(grep -v '^#' /opt/sapiosciences/local-exec-server.env | xargs)
```
And the file */opt/sapiosciences/local-exec-server.env* can be located anywhere in your system so long as it's readable to the Sapio BLS shell script.
The example .env text file should look like this:
```text
SapioNativeExecAPIKey="Your API Key. Please see README."
SapioNativeExecHost="host"
SapioNativeExecPort="8686"
SapioNativeExecTrustStoreData="use command 'base64 your_keystore_file' to generate this string"
SapioNativeExecTrustStorePassword="123456"
```

### Smoke Test
To run a smoke test, Go to ELN create a table of data
Create a column "x" and a column "y"
Enter a data suitable for linear regression data such as (1, 1), (2, 3), (4, 10), (5 15)

Create an advanced curve viewer widget as the next entry, select "Polynomial" regression and pick the X and Y columns.
If the configuration is correct, the linear regression will be performed.
You will also see in the app log indicating an attempt to connect to analytics server.

## Deployment of Sapio Analytics Cluster with Load balancing
There are multiple ways to deploy a Sapio Analytics Cluster with Load Balancing.
In this tutorial, we will show you a sample deployment using Kubernetes over AWS.

There are 3 Kubernetes YAML configuration files in this example.

### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytic-server-yq-github-dev-app
spec:
 selector:
   matchLabels:
     run: analytic-server-yq-github-dev-app
 template:
   metadata:
     labels:
       run: analytic-server-yq-github-dev-app
   spec:
    terminationGracePeriodSeconds: 630
    containers:
    - name: analytic-server-yq-github-dev-app
      image: sapiosciences/sapio_analytics_server_dev:<REPLACE_ME_WITH_TAG>
      imagePullPolicy: Always
      env:
      - name: SAPIO_EXEC_SERVER_API_KEY
        value: "<REPLACE_ME>"
      - name: SAPIO_EXEC_SERVER_KEYSTORE_PASSWORD
        value: "123456"
      - name: SAPIO_EXEC_SERVER_KEYSTORE_BASE64
        value: "<REPLACE_ME_TOO>"
      ports:
      - containerPort: 8686
      resources:
        requests:
          memory: "4096Mi"
          cpu: "1"
          ephemeral-storage: "20Gi"
```
In this file, we are defining a template of a single container. The image will be corresponding to the version of platform for your Sapio deployment.
You will have to replace the API key and keystore contents here using the instructions provided above.
Over at the resources section, you can freely modify the resource allocated per pod. However, the value should be no less than the provided value in the example.

### hpa.yaml
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
 name: analytic-server-yq-github-dev-app
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: analytic-server-yq-github-dev-app
 minReplicas: 1
 maxReplicas: 10
 metrics:
 - type: Resource
   resource:
     name: cpu
     target:
       type: Utilization
       averageUtilization: 50
 - type: Resource
   resource:
     name: memory
     target:
       type: Utilization
       averageUtilization: 70
```
In this file, we are defining the autoscaling policy for the deployment.
The autoscaling will be based on the CPU and memory utilization of the pods.
In the example above, we allow a total of 10 pods maximum to be created for analytic server usage horizontally in this cluster.
A new pod will be created if the average CPU utilization is above 50% or memory utilization is above 70%.
These settings might need to be fine-tuned based on the actual usage and budget of the analytic server.

A pod can be used to handle multiple requests simultaneously.
The pod will not immediately be disposed of when the usage is low, but is subject to a termination condition set in the deployment file.

**Please note that this is only defining the pod-level autoscaling and not the cluster-level autoscaling. This will not allocate any new instances in the cluster when the resources are full.**

If you would like to further scale the cluster instances, you will need to install AWS Autoscaler pod to kube system.
Please follow [this tutorial after completing the current setup](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)
Please note that allowing cluster instances to vary may increase the cost of the deployment.

### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
 name: analytic-server-yq-github-dev-app
 labels:
   run: analytic-server-yq-github-dev-app
 annotations:
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
  service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
 type: LoadBalancer
 ports:
 - port: 8686
   protocol: TCP
   targetPort: 8686
 selector:
   run: analytic-server-yq-github-dev-app
```

In the service file, we are defining how would the service be exposed to the public.
The correct load balance type is "NLB", which stands for "Network Load Balancer".
Depending on your exact needs, you may want to adjust the load balancer scheme for your particular security environment.

After you have created these files, you may want to create a CI/CD pipeline to deploy these configurations to your Kubernetes cluster.
For example, I have created a CI/CD pipeline to deploy our demo instance like follows:
```yaml
Name: codecatalyst-eks-yq-github-workflow
RunMode: SUPERSEDED
SchemaVersion: 1.0

# You can set the CI/CD to redeploy on changes to repo.
# Triggers:
#   - Type: PUSH
#     Branches:
#       - main
Actions:
  BuildBackend:
    Identifier: aws/build@v1
    Environment:
      Name: analytic-server-dev-test-env
      Connections:
        - Name: YOUR_ACCOUNT_ID
          Role: codecatalyst-eks-build-role
    Inputs:
      Sources:
        - WorkflowSource
      Variables:
        - Name: REPOSITORY_URI
          Value: YOUR REPO ACCESSIBLE BY CODE CATALYST.
        - Name: IMAGE_TAG
          Value: ${WorkflowSource.CommitId}
        - Name: CLUSTER_REGION
          Value: us-east-1
        - Name: CLUSTER_NAME
          Value: codecatalyst-github-analytic-server
        - Name: AMD_AMI_ID
          Value: AL2_x86_64
    Configuration:
      Steps:
        - Run: find Kubernetes/ -type f | xargs sed -i "s|\$AMD_AMI_ID|$AMD_AMI_ID|g"
        - Run: find Kubernetes/ -type f | xargs sed -i "s|\$CLUSTER_NAME|$CLUSTER_NAME|g"
        - Run: find Kubernetes/ -type f | xargs sed -i "s|\$CLUSTER_REGION|$CLUSTER_REGION|g"
        - Run: cat Kubernetes/*
    Outputs:
      Artifacts:
        - Name: Manifests
          Files:
            - "Kubernetes/*"
  DeployToEKS:
    DependsOn:
      - BuildBackend
    Identifier: aws/kubernetes-deploy@v1
    Environment:
      Name: analytic-server-dev-test-env
      Connections:
        - Name: YOUR_ACCOUNT_ID
          Role: codecatalyst-eks-deploy-role
    Inputs:
      Artifacts:
        - Manifests
    Configuration:
      Namespace: default
      Region: us-east-1
      Cluster: codecatalyst-sapio-analytic-server
      Manifests: Kubernetes/
```

However, when you simply want to refresh to newer version of image of the same tag, you don't need to re-run the pipeline.
Simply use the following command

```shell
kubectl rollout restart deployments/analytic-server-yq-github-dev-app
kubectl rollout status deployments/analytic-server-yq-github-dev-app
```
Your shell will be blocked on status command until the deployment is complete.
