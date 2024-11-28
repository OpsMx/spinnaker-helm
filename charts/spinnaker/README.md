# Spinnaker Chart

[Spinnaker](http://spinnaker.io/) is an open source, multi-cloud continuous delivery platform.

## Chart Details
This chart will provision a fully functional and fully featured Spinnaker installation
that can deploy and manage applications in the cluster that it is deployed to.

Redis and Minio are used as the stores for Spinnaker state.

For more information on Spinnaker and its capabilities, see it's [documentation](http://www.spinnaker.io/docs).

## Setup Instructions

### Prerequisites

- Kubernetes cluster 1.20 or later with at least 4 cores and 16 GB memory.
- Helm 3 is setup on the client system with 3.10.3 or later.

## Helm Chart supports two modes of Installations

1. Non-Gitops Method
2. Gitops Method

- Use below command to check if helm is installed or not
        
   ```console
   helm version
   ```
  If helm is not setup, follow <https://helm.sh/docs/intro/install/> to install helm.

## 1. Installing with Non-Gitops Method

- Add spinnaker helm repo to your local machine

   ```console
   helm repo add spinnaker https://opsmx.github.io/spinnaker-helm/
   ```

  Note: If spinnaker helm repo is already added, do a repo update before installing the chart

   ```console
   helm repo update
   ```
- Use below command to create the namespace

   ```console
   kubectl create namespace opsmx-oss
   ```

- Use below command to install the helm chart using Non-Gitops Method:

  ```console
  helm install oss-spin spinnaker/spinnaker -n opsmx-oss --timeout 600s
  ```
> **Tip**: For more information of changing the default values file please [check](charts/spinnaker/additionalinfo.md)
  

## 2. Installing with Gitops Method

- In this method all the halyard configuration will be centralised in Git Repository.
 
  -  Create an empty repo(called as "gitops-halyard") branch "main" as default, and clone to the local-machine.
     
  -  Clone the [repo](https://github.com/OpsMx/standard-gitops-halyard.git)

     ```console
     git clone https://github.com/OpsMx/standard-gitops-halyard.git
     ```

  -  Copy contents of the standard-gitops-halyard repo to the gitops-halyard repo

     ```console
     cp -r standard-gitops-halyard/* gitops-halyard # Replace "gitops-halyard" with your repo-name
     ```
  - cd to the newley created repo 
    
    ```console
    cd gitops-halyard
    ```

     ```console
     git add -A; git commit -m"Upgrade related changes";git push
     ```

- Create a K8s secret called opsmx-gitops-auth (Do not change the name of the secret)

- Copy the below file and update the gituser, gittoken, and gitcloneparam (this includes username, token, organisation and git-repository) values.

  Format of the secret: opsmx-gitops-auth’s yaml file

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://GIT_USERNAME:GIT_TOKEN@github.com/GIT_ORGANISATON/GIT_REPOSITORY.git
    gittoken: xxxxxxxxxxxx
    gituser: git-username
  type: Opaque
   ```

  After updating the secret values(username, token, organisation and git-repository) looks as below

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://jhon:ghbzceqed_adsfasdf@github.com/john/gitops-halyard.git
    gittoken: ghbzceqed_adsfasdf
    gituser: jhon
  type: Opaque
   ```

- Use below command to apply the secrets yaml
  
  ```console
  kubectl -n opsmx-oss apply -f secret.yaml
  ```

- Use below command to upgrade oss to gitops method.

  ```console
  helm install oss-spin spinnaker/spinnaker --set halyard.gitops.enabled=true --timeout 600s -n opsmx-oss
  ```

  **Note**: Make sure the same release name is used during installation.

## Accessing the Spinnaker

- Check the status of the pods by executing this command:

  ```console
  kubectl -n opsmx-oss get po
  ```

  Once all pods show "Running" or "Completed" status and Use port-forward command to access the Spinnaker:

  ```console
  kubectl -n opsmx-oss port-forward svc/spin-deck 9000  ## Keep running, it shows messages such as "Forwarding from 127.0.0.1:9000 -> 9000
  ```

  Now, open your browser and navigate to http://localhost:9000

## Securing Secret Credentails in the Halyard Git repo (Optional)

**Note**: Secrets in Halyard are plain-text, storing them as-is in Git repository is a security concern. Hence, we will replace all the Secrets/passwords in halyard config with a placeholder before committing them to the Git repository. During the halyard pod startup, these secrets are evaluated to their original value through an init container.

- Create one or more K8s secrets in the same namespace where Spinnaker is running, with your credentials.

  ```console
  kubectl -n opsmx-oss create secret generic <SecretName> --from-literal=<SecretKey>=<SecretValue> --from-file=myk8saccount-kube.config #File name becomes SecretKey
   ```

- Or, Use below yaml file (hal-secrets.yml) to create the secret
          
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: hal-secrets
  stringData:
    prodjenkinspwd: jenkinspassword
    gitopstoken: gittoken
    myk8saccount-kube.config: <kubeconfig-content>
  type: Opaque
  ```

  ```console
  kubectl -n opsmx-oss apply -f hal-secrets.yml
  ```

- Edit the hal config file (e.g: gitops-halyard/config) and update every password/confidential text as per the format here.

  - For passwords, the placeholder is

    ```console
    encrypted:<K8s-SecretName>:<SecretKey>
    ```

  - For kubeconfig and other confidential files, the placeholder is

    ```console
    encryptedFile:<K8s-SecretName>:<SecretKey>
    ```

    **Note**: The K8s-SecretName and SecretKey should be matching the secret created.

- A sample of the Hal config - before GitOps and after GitOps

  - Before GitOps - Sample:

    ```yaml
    github:
      enabled: true
      accounts: 
      - name: githubdemo_account
        username: "GITUSERNAME"
        token: "5cb4371fxxxxxxxxx5"
    ```

  - After GitOps - Sample:

    ```yaml
    github:
      enabled: true
      accounts: 
      - name: githubdemo_account
        username: "john"
        token: "encrypted:hal-secrets:gitopstoken"
    ```

    **Note**: After creating the secrets and updating the hal config file, you are now ready to commit the files to your remote git repository. Go ahead and complete it. Any changes you make in Halyard should be manually committed to Git repository; otherwise with every Halyard restart the changes will be gone and git repo content is the source of the truth for Gitops Halyard repo.

## License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
