# Spinnaker Chart

[Spinnaker](http://spinnaker.io/) is an open source, multi-cloud continuous delivery platform.

## Chart Details
This chart will provision a fully functional and fully featured Spinnaker installation
that can deploy and manage applications in the cluster that it is deployed to.

Redis and Minio are used as the stores for Spinnaker state.

For more information on Spinnaker and its capabilities, see it's [documentation](http://www.spinnaker.io/docs).

## Setup Instructions

### Prerequisites

- Kubernetes cluster 1.20 or later with at least 4 cores and 16 GB memory
- Helm 3 is setup on the client system with 3.10.3 or later

## Helm Chart supports two modes of Installations

1. Non-Gitops Method
2. Gitops Method

- Use below command to check if helm is installed or not
        
   ```console
   helm version
   ```
  If helm is not setup, follow <https://helm.sh/docs/intro/install/> to install helm.

## Installing the Chart in Non-Gitops Method

- Add spinnaker helm repo to your local machine

   ```console
   helm repo add spinnaker https://opsmx.github.io/spinnaker-helm/
   ```

  Note: If spinnaker helm repo is already added, do a repo update before installing the chart

   ```console
   helm repo update
   ```
- Use below command to install the helm chart using Non-Gitops Method:

  ```console
  helm install oss-spin spinnaker/spinnaker -n oss-spinnaker --timeout 600
  ```

## Installing the Chart in Gitops Method

**Note**: It is not possible to enable GitOps before installing Spinnaker, hence you need to get your OSS Spinnaker up and running before enabling GitOps for its Halyard configuration.

Prerequisites:

- You already have a running Spinnaker OSS, its Hal config is stored in the Halyard pod itself - no external backup in S3 or Git or anywhere else.
- You have admin access to Git Organization or useraccount to create Git repositories.

**Procedure for Enabling GitOps for Halyard**

- Store the Halyard configuration in a Git repository with a designated directory structure. This is explained in the section Configure Hal config Git repo.
- Storing Halyard Secrets in the Git repo.
- Create a K8s secret - opsmx-gitops-auth to be used by Halyard after enabling GitOps.
- Upgrade Spinnaker and Halyard manifest that it is GitOps enabled by performing Helm upgrade. This is explained in the section Upgrade Spinnaker with GitOps.
- Configure Hal config Git repo

**Note**: Spinnaker Halyard is assumed to be running with no GitOps for its configuration, with these steps we are enabling it for GitOps.

## Below are the steps to be followed

- Login to the halyard pod and take backup of the hal config

   ```console
   kubectl -n <namespace> exec -it <halyard-pod-name> -- bash
   ```

   ```console
   cd /home/spinnaker
   ```

- Create the hal backup using the below command

   ```console
   hal backup create
   ```
   The command above will output the log where you will find a line something like below
   
   Successfully created backup at location: /home/spinnaker/halyard-2023-02-09_11-24-45-662Z.tar 

- Copy the tar to other file

   ```console
   cp /home/spinnaker/halyard-2023-02-09_11-24-45-662Z.tar halbkup.tar
   ```
   exit #Return to local-machine

- Copy the hal backup to local-machine

   ```console
   kubectl -n <namespace> cp <halyard-pod-name>:/home/spinnaker/halbkup.tar ./hal.tar
   ```

-  A git-repository is required to store your Hal config files. Create an empty repo in your github/bitbucket/gitlab organisation, and clone it to your local-machine.

#Example refers to github.com, but it can be any remote #git-repository. Sample repo is https://github.com/OpsMx/oss-hal.git.

  `git clone https://github.com/OpsMx/oss-hal.git`

-  Untar the backup file (hal.tar) inside the git-repository directory. Note, the hal config files should go inside the git-repo directory (not anything else).

- Command to untar the file

  ```console
  tar -xvf hal.tar -C <git-local-dir>`
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

-  Create the halyard.yaml file in the same directory where the hal config file is present. Content of the file is as below:

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
Edit the hal config file and update the deploymentEnvironment.location value with literal string SPINNAKER_NAMESPACE 
#cd oss-hal/
vi config

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

Now update the confidential texts as per the next section [Storing Halyard Secrets in the Git repo] and commit the changes to remote git repository
# NOTE: Do not commit before changing the confidential texts with encrypted format


$ git add .
$ git commit “Initial commit of Hal config”
$ git push

Storing Halyard Secrets in the Git repo
Note: Secrets in Halyard are plain-text, storing them as-is in Git repository is a security concern. Hence, we will replace all the Secrets/passwords in halyard config with a placeholder before committing them to the Git repository. During the halyard pod startup, these secrets are evaluated to their original value through an init container.

Create one or more K8s secrets in the same namespace where Spinnaker is running, with your credentials.
kubectl -n <namespace> create secret generic <SecretName> --from-literal=<SecretKey>=<SecretValue> --from-file=myk8saccount-kube.config #File name becomes SecretKey

Or, have a yaml manifest file (hal-secrets.yml) to create the secret
```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: hal-secrets
stringData:
  prodjenkinspwd: <password>
  gitopstoken: <git-token>
  myk8saccount-kube.config: <kubeconfig-content>
 ```

kubectl apply -f hal-secrets.yml

Edit the hal config file (e.g: oss-hal/config) and update every password/confidential text as per the format here.
For passwords, the placeholder is 
encrypted:<K8s-SecretName>:<SecretKey>

For kubeconfig and other confidential files, the placeholder is 
encryptedFile:<K8s-SecretName>:<SecretKey>

Note: The K8s-SecretName and SecretKey should be matching the value you have done in the previous step i.e step 1

A sample of the Hal config - before GitOps and after GitOps 
Before GitOps - Sample:
    github:
      enabled: true
      accounts: 
      - name: opsmxdemo_account
        username: "GITUSERNAME"
        token: "5cb4371fxxxxxxxxx5"


After GitOps - Sample:
    github:
      enabled: true
      accounts: 
      - name: opsmxdemo_account
        username: "GITUSERNAME"
        token: "encrypted:<K8s-SecretName>:<SecretKey>"


Note: After creating the secrets and updating the hal config file, you are now ready to commit the files to your remote git repository. Go ahead and complete it.
Create a K8s secret - opsmx-gitops-auth
Note: The secret opsmx-gitops-auth stores the Git-repo URL and credential to access the repo in Halyard pod’s init script. It is important to keep the secret name as-is.

Copy the below file and update the gituser, gittoken, and gitparam (this includes username, token, organisation and git-repository) values.

Format of the secret: opsmx-gitops-auth’s yaml file


apiVersion: v1
stringData:
  gitcloneparam: https://GIT_USERNAME:GITTOKEN@github.com/GIT ORGANISATON/GIT_REPOSITORY.git  # Use your Git clone Https URL, Update the gitusername,gittoken,organisation and git repository
  gittoken: xxxxxxxxxxxx # Update the Git token 
  gituser: git-username  #Update the Username
kind: Secret
metadata:
  name: opsmx-gitops-auth
type: Opaque


           Below is a sample for bitbucket secret values: 

apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: opsmx-gitops-auth
stringData:
  gitcloneparam: https://BITBUCKETUSERNAME:xxxxx@bitbucket.org/BITBUCKETUSERNAME/BITBUCKETREPO.git
  gittoken: BITBUCKET TOKEN# Update the Git token 
  gituser: BITBUCKET USERNAME  #Update the Username


           Below is a sample for enterprise bitbucket:
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: opsmx-gitops-auth
stringData:
  gitcloneparam: https://BITBUCKETUSERNAME:BITBUCKETTOKEN@<enterprisebitbuckethostname>/scm/projectName/BITBUCKETREPO.git
  gittoken: BITBUCKET TOKEN# Update the Git token 
  gituser: BITBUCKET USERNAME  #Update the Username


Upgrade Spinnaker with GitOps
Note: Git repository should have the hal config content, otherwise Halyard pod will fail to start. Helm upgrade will update the Halyard init script (configmap) and Halyard init pod making it ready for GitOps.


Clone the OpsMx’s spinnaker-helm repo and modify the values.yaml file for helm upgrade.
$ git clone https://github.com/opsmx/spinnaker-helm.git -b oss-gitops
$ cd spinnaker-helm/

Create a copy of charts/spinnaker/values.yaml file and edit it to ensure GitOps is enabled
halyard.gitops.enabled: true
 
Upgrade existing Spinnaker instance with helm command. Ensure the release name and namespaces are matching your initial helm install parameters.
$ helm --debug upgrade <release-name> charts/spinnaker/ -f values.yaml  --timeout=800s -n <namespace>

After the above steps, Halyard will restart and will pull the changes from the Git repository. From this point onwards, any changes you make in Halyard should be manually committed to Git repository; otherwise with every Halyard restart the changes will be gone and git repo content is the source of the truth for Halyard config.

