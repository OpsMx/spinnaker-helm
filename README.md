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

## Installing with Non-Gitops Method

- Non-Gitops Method will install the Open Source Spinnaker.

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
  
## Accessing the Spinnaker

- Check the status of the pods by executing this command:

  ```console
  kubectl -n opsmx-oss get po
  ```

  Once all pods show "Running" or "Completed" status and Use port-forward command to access the Spinnaker:

  ```console
  kubectl -n opsmx-oss port-forward svc/spin-deck 9000  ## Keep running, it shows messages such as "Forwarding from 127.0.0.1:8080 -> 8080
  ```

  Now, open your browser and navigate to http://localhost:9000

## Gitops Method

- In this method all the halyard configuration will be centralised in Git Repository.

**Note**: It is not possible to enable GitOps before installing Spinnaker, hence you need to get your OSS Spinnaker up and running before enabling GitOps for its Halyard configuration and user need to have admin access to Git Organization or useraccount to create Git repositories.
 
  -  Create an empty repo(called as "gitops-halyard") and clone to the local-machine.
     
  -  Clone https://github.com/OpsMx/standard-gitops-halyard.git

  -  Copy contents of the standard-isd-repo to the gitops-repo created above using:

     ```console
     cp -r standard-gitops-halyard/* gitops-halyard # Replace "gitops-halyard" with your repo-name
     ```

     ```console
     git add -A; git commit -m"Upgrade related changes";git push
     ```

- Create a K8s secret - opsmx-gitops-auth

  **Note**: The secret opsmx-gitops-auth stores the Git-repo URL and credential to access the repo in Halyard pod’s init script. It is important to keep the secret name as-is.

- Copy the below file and update the gituser, gittoken, and gitcloneparam (this includes username, token, organisation and git-repository) values.

  Format of the secret: opsmx-gitops-auth’s yaml file

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://GIT_USERNAME:GIT_TOKEN@github.com/GIT_ORGANISATON/GIT_REPOSITORY.git  # Use your Git clone Https URL, Update the gitusername,gittoken,organisation and git repository
    gittoken: xxxxxxxxxxxx # Update the Git token 
    gituser: git-username  #Update the Username
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

- Use below command to upgrade oss to gitops method.

  ```console
  helm upgrade oss-spin spinnaker/spinnaker --set halyard.gitops.enabled=true --timeout 600s -n opsmx-oss
  ```

  **Note**: Make sure the same release name is used during installation.

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
  kubectl apply -f hal-secrets.yml -n opsmx-oss
  ```

- Edit the hal config file (e.g: gitopsdir/config) and update every password/confidential text as per the format here.

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
      - name: opsmxdemo_account
        username: "GITUSERNAME"
        token: "5cb4371fxxxxxxxxx5"
    ```

  - After GitOps - Sample:

    ```yaml
    github:
      enabled: true
      accounts: 
      - name: opsmxdemo_account
        username: "GITUSERNAME"
        token: "encrypted:<K8s-SecretName>:<SecretKey>"
    ```

    **Note**: After creating the secrets and updating the hal config file, you are now ready to commit the files to your remote git repository. Go ahead and complete it. Any changes you make in Halyard should be manually committed to Git repository; otherwise with every Halyard restart the changes will be gone and git repo content is the source of the truth for Halyard config.
