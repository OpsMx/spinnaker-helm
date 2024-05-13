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

1. Non-Gitops Method: This is the normal mode of Spinnaker Halyard, wherein all the configuration is inside Halyard
2. Gitops Method: The Spinnaker configuration is stored in a git-repo, and the halyard syncs with the repo. While it involves an extra setup, during changes/upgrades, we can see exactly what is changing as all changes, include spinnaker configuration can be routed via git-PRs.

- Use below command to check if helm is installed or not
        
   ```console
   helm version
   ```
  If helm is not setup, follow <https://helm.sh/docs/intro/install/> to install helm.

## Installing with Non-Gitops Method

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
  
## Accessing Spinnaker after installation

- Check the status of the pods by executing this command:

  ```console
  kubectl -n opsmx-oss get po
  ```

  Once all pods show "Running" or "Completed" status and Use port-forward command to access the Spinnaker:

  ```console
  kubectl -n opsmx-oss port-forward svc/spin-deck 9000  ## Keep running, it shows messages such as "Forwarding from 127.0.0.1:9000 -> 9000
  ```

  Now, open your browser and navigate to http://localhost:9000

Alternatively, you can route traffic via Ingress/LB to the spin-deck and spin-gate services. Details are **here**.

## Gitops Method

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

- Use minimal-values.yaml inside charts/spinnaker folder Ex: https://github.com/OpsMx/spinnaker-helm/blob/1.33.0/charts/spinnaker/minimal-values.yaml

- Update the gitorg, gitrepo, gituser, dynaaccrepo, gittoken and save the minimal-values.yaml

- Use below command to install oss in gitops method.

  ```console
  helm install oss-spin spinnaker/spinnaker -f minimal-values.yaml --timeout 600s -n opsmx-oss
  ```

  **Note**: Make sure the same release name is used during installation.

## Securing Secret Credentails in the Halyard Git repo (Optional)

**Note**: Secrets in Halyard are plain-text, storing them as-is in Git repository is a security concern. We can replace all the Secrets/passwords in halyard config with a placeholder before committing them to the Git repository. During the halyard pod startup, these secrets are evaluated to their original value through an init container.

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
    
## Support
Limited support is available on Spinnaker Slack (spinnakerteam.slack.com).

Channel: opsmx
