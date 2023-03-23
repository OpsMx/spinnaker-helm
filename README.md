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

- Use below command to install/upgrade to the latest verion of Spinnaker using the helm chart.

  ```console
  helm upgrade --install oss-spin spinnaker/spinnaker --set halyard.spinnakerVersion=1.29.3 --timeout 600s -n opsmx-oss
  ```

  **Note**: In the above command replace the 1.29.3 with appropriate spinnaker version so that it will install that particular version.

## Converting OSS to Gitops Method

- In this method all the halyard configuration will be centralised in Git Repository.

**Note**: It is not possible to enable GitOps before installing Spinnaker, hence you need to get your OSS Spinnaker up and running before enabling GitOps for its Halyard configuration and user need to have admin access to Git Organization or useraccount to create Git repositories.

**Below are the steps to be followed**

- Login to the halyard pod and take backup of the hal config

  ```console
  kubectl -n <namespace> exec -it <halyard-pod-name> -- bash
  ```

  ```console
  cd /home/spinnaker
  ```

  ```console
  hal backup create
  ```
   
  The above command will output the log where you will find a line something like below
   
  `Successfully created backup at location: /home/spinnaker/halyard-2023-02-09_11-24-45-662Z.tar` 

- Copy the tar to other file

  ```console
  cp /home/spinnaker/halyard-2023-02-09_11-24-45-662Z.tar halbkup.tar
  ```

- Copy the hal backup to local-machine

  ```console
  kubectl -n <namespace> cp <halyard-pod-name>:/home/spinnaker/halbkup.tar ./hal.tar
  ```

- A git-repository is required to store your Hal config files. 
 
  -  Create an empty repo in your organisation(github/bitbucket/gitlab).

  -  Clone it to your local-machine.
     
     `git clone https://github.com/ORGANISATION/REPO_NAME.git`

- Untar the backup file (hal.tar) to the git-repository directory.

  ```console
  tar -xvf hal.tar -C <REPO_NAME>
  ```

- The directory structure looks like this

  ```dir
  oss-hal/
  ├── config
  ├── default/
  │   ├── profiles/
  │   │   ├── front50-local.yml
  │   │   ├── gate-local.yml
  │   └── service-settings/
  │       ├── deck.yml
  │       ├── redis.yml
  ├── halyard.yaml #Create this file using the content down below
  └── README.md
  ```

- Create the halyard.yaml file in the same directory where the hal config file is present. Content of the file is as below:

  ```yaml
  server:
    port: 8064
  grpc:
    enabled: false
  halconfig:
    filesystem:
      path: ~/.hal/config
  spinnaker:
    artifacts:
      debian: https://dl.bintray.com/spinnaker-releases/debians
      docker: gcr.io/spinnaker-marketplace
    config:
      input:
        gcs:
          enabled: true
        writerEnabled: false
        bucket: halconfig
  management:
    endpoint:
      shutdown:
        enabled: true
    endpoints:
      web:
        exposure:
          include: shutdown, env, conditions, resolvedEnv, beans,health
  backup:
    google:
      enabled: false
  retrofit:
    logLevel: BASIC
  ```

- Edit the hal config file and update the deploymentEnvironment.location value with literal string SPINNAKER_NAMESPACE 

  Here is how it looks,

  ```yaml
  deploymentEnvironment:
    size: "SMALL"
    type: "Distributed"
    accountName: "default"
    imageVariant: "SLIM"
    updateVersions: true
    consul:
      enabled: false
    vault:
      enabled: false
    location: SPINNAKER_NAMESPACE #Keep this text
  ```

  **NOTE**: Do not commit any sensitive information(i.e credentials) to git repo.

  ```console
  git add -A; git commit -m"Upgrade related changes";git push
  ```

- Create a K8s secret - opsmx-gitops-auth

  **Note**: The secret opsmx-gitops-auth stores the Git-repo URL and credential to access the repo in Halyard pod’s init script. It is important to keep the secret name as-is.

- Copy the below file and update the gituser, gittoken, and gitparam (this includes username, token, organisation and git-repository) values.

  Format of the secret: opsmx-gitops-auth’s yaml file

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://GIT_USERNAME:GIT_TOKEN@github.com/GIT_ORGANISATON/GIT_REPOSITORY.git  # Use your Git clone Https URL, Update the        g  itusername,gittoken,organisation and git repository
    gittoken: xxxxxxxxxxxx # Update the Git token 
    gituser: git-username  #Update the Username
  type: Opaque
   ```

  Sample reference for bitbucket secret: 

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://BITBUCKET_USERNAME:BITBUCKET_TOKEN@bitbucket.org/BITBUCKET_USERNAME/BITBUCKET_REPO.git
    gittoken: USER_TOKEN  # Update the token 
    gituser: USERNAME   # Update the Username
  type: Opaque
  ```

  Sample reference for enterprise bitbucket secret:
  
  ```yaml
  apiVersion: v1
  kind: Secret
  type: Opaque
  metadata:
    name: opsmx-gitops-auth
  stringData:
    gitcloneparam: https://USERNAME:TOKEN@<enterprisebitbuckethostname>/scm/projectName/BITBUCKET_REPO.git
    gittoken: USER_TOKEN  # Update the token 
    gituser: USERNAME  # Update the Username
  ```

- Use below command to upgrade oss to gitops method.

  ```console
  helm upgrade oss-spin spinnaker/spinnaker --set halyard.gitops.enabled=true --timeout 600s -n oss-spinnaker
  ```

  **Note**: Make sure the same release name is used during installation.

## Securing Secret Credentails in the Halyard Git repo (Optional)

**Note**: Secrets in Halyard are plain-text, storing them as-is in Git repository is a security concern. Hence, we will replace all the Secrets/passwords in halyard config with a placeholder before committing them to the Git repository. During the halyard pod startup, these secrets are evaluated to their original value through an init container.

- Create one or more K8s secrets in the same namespace where Spinnaker is running, with your credentials.

  ```console
  kubectl -n <namespace> create secret generic <SecretName> --from-literal=<SecretKey>=<SecretValue> --from-file=myk8saccount-kube.config #File name       becomes SecretKey
   ```

- Or, Use below yaml file (hal-secrets.yml) to create the secret
          
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: hal-secrets
  stringData:
    prodjenkinspwd: <password>
    gitopstoken: <git-token>
    myk8saccount-kube.config: <kubeconfig-content>
  type: Opaque
  ```

  ```console
  kubectl apply -f hal-secrets.yml -n <namespace>
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
