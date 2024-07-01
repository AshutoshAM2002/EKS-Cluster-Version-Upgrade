_**EKS Upgrade can not be rolled back. So make sure you follow all the steps mentioned in the Doc. If you face any problem which is not mentioned in the Doc then please inform Doc-Auther about the same to help us to make our Doc up to date.**_

***
## 1. Things needs to be checked before and after cluster upgrade. 
- Verify that all applications are running fine before EKS upgrade-
  ```console
  kubectl get pods -A
  kubectl logs podname -n namespace-name
  kubectl describe pod podname -n namespace-name
  kubectl get ing -A
  kubectl get virtualservices -A
  ```

## 2. Look for the deprecated and removed resources, objects and APIs
- Run `kubent` to get the information, Install kubent from [HERE](https://github.com/doitintl/kube-no-trouble) if haven't already.

![image](https://user-images.githubusercontent.com/83774016/228209811-a257ddf2-300f-4f71-92e9-adfcad67c86a.png)
- In the above result, you need to fix the `apiVersion` and `syntax`-changes(if any) in the respective objects. Google for the latest apiVersion and Syntax.

- **About Pod Security Policy (PSP)**
  - If any application/pod is using PSP and it is deployed using HELM-Chart then uninstall it. Make sure to take backup of these releases - 
    - take backup using below commands - 
      ```bash
      helm ls -A
      helm get manifest releasename -n namespacename > releasename_namespacename.yaml
      ```
    - Uninstall helm chart 
      ```bash
      helm ls -A
      helm uninstall releasename -n namespacename
      ```
  - eks.privileged will be removed automatically as it is not installed using helm chart.

- **update every helmchart to its latest version**
  - Check all helm releases using `helm ls -A`
  - Do Google Search for every helm release except if the release is for an application.
  - Check the application and eks comaptibility. 
  - For `aws-load-balancer-controller` uninstall this chart and re-install the chart with latest image. Upgrading `aws-load-balancer-controller` helmchart will cause other application helm deployemnt and might throw `mservice.elbv2.aws.k8s service not found` issue.


- If the chart is present in the remote-repository like GitHub then clone it to local and fix the changes and upgrade the helmchart.
- You can pull the helmchart in current directory using below command -
  ```console
   helm list -A
   helm pull chartname --untar
  ```
![image](https://user-images.githubusercontent.com/83774016/228220528-183d7094-d0fd-45db-b97b-7aa75459c8ef.png)
  
  - use `replicaCount=2` in `helm upgrade` command for critical releases.
  - For eg: `helm upgrade release chart -n namespace --atomic --set replicaCount=2`
  - After helm update check logs of newly generated pods. 
    - Pod Logs - `kubectl logs -f podName -n nameSpace` 


- Now Again run the `kubent` command to check any deprecated resources or APIs. If any then fix them also.

- If the PSP still persist then for application then **Update their namespace and add below labels**

  ```yaml
  metadata:
    labels:    
      pod-security.kubernetes.io/audit: restricted
      pod-security.kubernetes.io/warn: restricted
  ```

  - this will show some warning, don't panic and `DO NOT` delete any pod or do not uninstall any chart.

- For HorizontalPodAutoscaler (HPA/ hpa.yaml) :
  - Change apiVersion from `autoscaling/v2beta1` to `autoscaling/v2`

    ```yaml
    apiVersion: autoscaling/v2
    ```

  - Change resource syntax under spec **FROM**

    ```yaml
    - type: Resource
      resource:
        name: cpu
        targetAverageUtilization: {{ .Values.microservices.chat.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.microservices.chat.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        targetAverageUtilization: {{ .Values.microservices.chat.autoscaling.targetMemoryUtilizationPercentage }}
    ```

    - **TO BELOW syntax**


    ```yaml
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization 
          averageUtilization: {{ .Values.microservices.chat.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.microservices.chat.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization         
          averageUtilization: {{ .Values.microservices.chat.autoscaling.targetMemoryUtilizationPercentage }}
    ```


## 3. Update EKS AddOns Manually From AWS Management Console.
- Login to AWS
- Search EKS and go inside your Cluster
- Select Add-ons tab
- Select First Add-on and click on `Update Version` 

![image](https://user-images.githubusercontent.com/83774016/228215533-22a19fa1-79d5-4a5c-9748-8088aa4b8e0c.png)

- Now from the `Version` Drop-Down, select the next minor and least patch version. For eg - 
  * v1.13.0-eksbuild.2  ⟶ v1.14.1-eksbuild.1  = WRONG :x: 
  * v1.13.0-eksbuild.2  ⟶ v1.14.0-eksbuild.1  = RIGHT :heavy_check_mark: 

![image](https://user-images.githubusercontent.com/83774016/228216546-52a813d3-a6b6-43af-9048-0f1e3623ed40.png)

- If the current minor is the latest minor then select the latest patch version.
  * v1.12.1-eksbuild.1  ⟶ v1.12.1-eksbuild.2 ______= WRONG :x: 
  * v1.12.1-eksbuild.1  ⟶ v1.12.6-eksbuild.1 (latest) = RIGHT :heavy_check_mark: 

<img width="634" alt="image (10)" src="https://user-images.githubusercontent.com/83774016/228218112-038076c4-36d5-48c6-a12c-7255f0c0b43e.png">


- Expand 'Optional configuration settings' and choose `None` and do `Save Changes` 
- If Update Fails then next time choose `Override` other setting will be same as above.

![image](https://user-images.githubusercontent.com/83774016/228219835-c81a0831-c450-4951-88b4-fc1d67cb717a.png)


## 4. Updating Master Node
- :warning: DO NOT UPDATE MASTER NODE UNTILL ALL DEPRECATED Resources & APIs ARE FIXED. Atlease DON'T UPDATE IN PRODUCTION.
- Now Goto AWS Management Console and search EKS.
- Click on `Update Now` showing next to the cluster version this will Update the Master Node.
  - This Update may take 5-10 minutes.
![image](https://user-images.githubusercontent.com/83774016/228230195-e0a9e6e2-1dd1-4eb7-a7f8-078faa953aae.png)


- After Updating Master Node access it using aws-cli and check the serverVersion.
  ```console
  aws eks update-kubeconfig --name clusterName --region regionCode
  kubectl version --output=yaml
  ```

## 5. Now Update the Worker Nodes
  - Go inside you cluster from AWS Management Console
  - Goto Compute Tab
  - Click on `Update Now` showing next to the `AMI release version` **in Node Groups** this will take 20-30 minutes based on the cluster size.
  - **It is recommended to Update Application/Critical NodeGroup first.**
  
![image](https://user-images.githubusercontent.com/83774016/228231784-56d89491-0e17-4ac7-815d-a97332371c09.png)


  - After Update when status changes to Active, check the Update-History of NodeGroup. It should be successfull
  
![image](https://user-images.githubusercontent.com/83774016/228232604-d9f9e1d2-2d6c-458d-809c-6dee473b30f7.png)
 
  - Check the nodes version from aws-cli. `kubectl get nodes`
  - The version of all nodes must be same and up to date.
***
  - If any node has old version then -
    - Increase the maxSize in AutoScalingGroup and re-update the Node-Group.
    - After 1-2 minutes remove the EC2 instace (having old node version) from AWS Management Console _(Filter EC2 instance by node-IP to get correct EC2 instance)_
  ```console
  kubectl get nodes -owide
  ```
<img width="773" alt="image (11)" src="https://user-images.githubusercontent.com/83774016/228249255-d4ff38f2-f718-4f0e-aa4e-3b2a9d8a6337.png">


  - :warning: **If Update is Successful and there is no option to re-update the Node-Group then inform `Seniors` about it As Soon As Possible .**
  - **DO NOT** change the Instance Type. rather you can increase the instance count from AutoScalingGroup to make more resources available. Changing the instance type will cause the application down.
  - Check the pods and thier logs.
  - If error is related to Networking then -
    - Go inside any of the running pod by `kubectl exec -it podName -n nameSpace /bin/sh` and execute below command to check connectivity.
    - Output could be anything (404, 403) but there should be no timeout.
      ```console
      curl -k https://172.20.0.1:443/api/v1/namespaces/kube-system/configmaps/extension-apiserver-authentication
      ```
    - If above command gives timeout then for troubleshooting purpose change the cluster security group rule (inbound & outbound) to `All traffic : anywhere`
    - If the issue persist then check the Service (`kubectl get svc -A`) and ServiceAccount (`kubectl get sa -A`).

***
## 6. EKS Update is Complete.
- Now you can test the deployment by deleting pods.
- Deploy an Helloworld application to check the working.

***
