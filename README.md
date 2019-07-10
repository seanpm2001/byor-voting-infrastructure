# BYOR-VotingApp [infrastructure]

Welcome to the repository for the infrastructure setup of **BYOR-VotingApp**!

You can find more information about the BYOR-VotingApp in the web-app Github [repository](https://github.com/thoughtworks/byor-voting-web-app).


#### Table Of Contents

[Running BYOR-VotingApp locally](#running-byor-votingapp-locally)

[Deploy BYOR-VotingApp to AWS Lambda](#deploy-byor-votingapp-to-aws-lambda)
-   [Prerequisites](#prerequisites-1)
-   [Setting up AWS](#setting-up-aws-1)
-   [Deploying the application](#deploying-the-application-1)
-   [Updating the application](#updating-the-application-1)

[Deploy BYOR-VotingApp to Kubernetes](#deploy-byor-votingapp-to-kubernetes)
-   [Provisioning AWS EKS Kubernetes cluster](#provisioning-aws-eks-kubernetes-cluster)
    -   [Setting up AWS](#setting-up-aws-2)
    -   [Provisioning with Terraform](#provisioning-with-terraform)
-   [Setting up an already provisioned cluster](#setting-up-an-already-provisioned-cluster)
-   [Deploying the application](#deploying-the-application-2)
-   [Updating the application](#updating-the-application-2)
-   [HOWTOs](#howtos)
    -   [Access Kubernetes Dashboard](#access-kubernetes-dashboard)
    -   [Validating certificate issuer](#validating-certificate-issuer)
    -   [Generating certificates with Let's encrypt](#generating-certificates-with-lets-encrypt)
    -   [How to manage secrets](#how-to-manage-secrets)

[How to contribute to the project](#how-to-contribute-to-the-project)
## Running BYOR-VotingApp locally

1) install [Docker](https://www.docker.com/get-started)
1) open the terminal
1) clone the project
    ```shell
    git clone https://github.com/thoughtworks/byor-voting-web-app.git
    ```
1) move into the project folder
    ```shell
    cd byor-voting-web-app
    ```
1) :warning: **[*TODO*]** startup web app, server, and a local MongoDB
    ```shell
    docker-compose up 
    ```
1) access the application on [http://localhost:4200](http://localhost:4200)

> Please refer to [CONTRIBUTING.md](CONTRIBUTING.md) for more options on running the web app locally.

> Please refer to [BYOR-VotingApp \[server\]](https://github.com/thoughtworks/byor-voting-server) Github repository for more options on running the server locally and connect to a MongoDB database.

## Deploy BYOR-VotingApp to AWS Lambda

### Prerequisites

-    install [GNU Bash](https://www.gnu.org/software/bash/)
-    install [GNU Make](https://www.gnu.org/software/make/)
-    install [npm](https://www.npmjs.com/get-npm)
-    install [AWS Command Line interface](https://aws.amazon.com/cli/)
-    clone **VotingApp [web-app]**
```shell
git clone https://github.com/thoughtworks/byor-voting-web-app.git
```
-    clone **VotingApp [server]**
```shell
git clone https://github.com/thoughtworks/byor-voting-server.git
```
-    clone **VotingApp [infrastructure]**
```shell
git clone https://github.com/thoughtworks/byor-voting-infrastructure.git
```

### Setup an AWS account

1. sign-in or create a new account in [AWS](https://aws.amazon.com) 
1. go to `IAM` -> `Users`
1. create a new user to be used for deployment
1. create an `Access key` for the user and download it
   > keep note of the `Access key ID` and `Secret access key` contained inside the file, they will be asked later by the deployment script.


### Setup MongoAtlas account and database
1. sign-in or create a new account in [MongoAtlas](https://www.mongodb.com/cloud/atlas)
1. create a new database using a lowercase name without spaces (e.g. production) and use `migrations` for the collection name
1. go to `Database Access`
1. create a user setting `add default privileges` to `readWrite` for the database defined above 
1. go to `Project`, click on `connect` and then on `Connect your application`
1. select `Node.js` from `Driver` dropdown
   > keep note of the `Connection string only` (replace the `<password>` with the user password defined above), it will be asked later by the deployment script
1. go to `Project`, click on `...` and then on `Command Line Tools`
   > keep note of the `--host` parameter value from the `mongorestore` example, it will be asked later by the deployment script

### Deploy to AWS (single installation)
Run the `deploy_to_lambda.sh` script passing the name of the installation. Use a lowercase name, without spaces  (e.g. production).
If you want to deploy to several installation targets at once pass them as a comma separated list (e.g. test,production).

```shell
cd byor-voting-infrastructure
aws/deploy_to_lambda.sh <installation_name1[,installation_nameX]>
```

The first time you run the script you will be asked to enter several informations. Afterwards, all the parameters will be stored inside the `config/byor_<installation_name>.sh` file. If you change some of the configuration values later, please either delete the file and let the script ask you again them, or update the file manually.

Requested values:
-    AWS access key id [`AWS_ACCESS_KEY_ID`]: the value of the `Access key ID` field contained downloaded file from point 4 of AWS instructions
-    AWS secret access [`AWS_SECRET_ACCESS_KEY`]: the value of the `Secret access key` field contained downloaded file from point 4 of AWS instructions
-    AWS region [`AWS_REGION`]: the AWS region where you want to host the application 
-    MongoDB connection string [`MONGO_HOME`]: the local path of your mongodb installation home (required if you want to perform backups)
-    MongoDB database [`MONGO_HOST`]: the value from point 7 in the MongoAtlas instructions
-    MongoDB host [`MONGO_USER`]: the value from point 4 in the MongoAtlas instructions
-    MongoDB username [`MONGO_PWD`]: the value from point 4 in the MongoAtlas instructions
-    MongoDB password [`MONGO_AUTH_DB`]: the default value (`admin`) is usually the right one
-    MongoDB admin database [`MONGO_URI`]: he value from point 6 in the MongoAtlas instructions

The script will create S3 buckets with the following naming conventions:
-   `<installation_name>--byor:` contains the build-your-own-radar SPA (single page application)
-   `<installation_name>--byor-voting`: contains the byor-voting-server deployed as lambda
-   `<installation_name>--byor-voting-web-app`: contains the byor-voting-web-app SPA (single page application)

The script will configure `<installation_name>--byor` and `<installation_name>--byor-voting-web-app` buckets as [static contents' web server](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html).

### Updating the application
To update the **web-app** or the **server**, just execute again the `aws/deploy_to_lambda.sh` script as above.


## Deploy BYOR-VotingApp to Kubernetes

### Provisioning AWS EKS Kubernetes cluster

-    install [AWS Command Line interface](https://aws.amazon.com/cli/)
-    install [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
-    install [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html)
-    install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

#### Setting up AWS

1)    login into AWS console:
        -    create [programmatic access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) for provisioning resources using Terraform and attach the right EC2 policies.
        -   configure AWS [cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) to interact with AWS
        -    [create EC2 keypair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
        -    [create AMI](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/tkv-create-ami-from-instance.html)
        -    create S3 bucket to store Terraform state file
        >   if you want you can use [aws/create_s3_bucket.sh](aws/create_s3_bucket.sh) script to perform the operation
        -    set [permission on S3](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-configure-bucket.html) to ensure to be accessible only by your organization
1)    inside [terraform.tf](terraform.tf)
        -    replace ``<terraform-state-storage>`` with the S3 bucket you create above
        -    replace ``<region>`` with the AWS region of your choice
1)    inside [variable.tf](variable.tf)
        -    replace ``<AMI-ID>`` with the AMI ID
        -    replace ``<keypair name>`` with the Keypair name
        -    customize other settings for eks (e.g. node_instance_type) based on your needs.
1)    inside [terraform.tfvars](terraform.tfvars) replace ``<aws_access_key>``, ``<aws_secret_key>``,``<aws_zones>`` with your AWS settings

#### Provisioning with Terraform

1)    open terminal and login into AWS
1)    move into the **VotingApp [infrastructure]** project folder
1)    duplicate terraform template files to replace sample variables:
        ```shell
        cp terraform.tf.sample terraform.tf
        cp terraform.tfvars.sample terraform.tfvars
        cp variables.rf.sample variables.rf
        ```
1)    if this is the first time you run terraform, execute:
        ```shell
        terraform init
        ```
1)    review the plan outputs:
        ```shell
        terraform plan
        ```
1)    if everything looks good, run:
        ```shell
        terraform apply
        ```
1)    if everything looks good, run:
        ```shell
        terraform apply
        ```
1)    to install Isto, Cert-Manager, Kiali secrets, and Let's encrypt secrets:
        ```shell
        source k8s/k8s_setup.sh
        ```
1)    if you want to delete all the resources managed by terraform, run:
        ```shell
        terraform destroy
        ```

## Setting up an already provisioned Kubernetes cluster

1)    set the ``KUBECONFIG`` context
        ```shell
        export KUBECONFIG=<path-to-kubeconfig>
        echo "export KUBECONFIG=${KUBECONFIG}" >> ${HOME}/.bashrc
        ```
1)    if you don't have already installed Isto, Cert-Manager, Kiali secrets, and Let's encrypt secrets, you can do it running:
        ```shell
        k8s/k8s_setup.sh
        ```


## Deploying the application

1)    install [helm](https://helm.sh/docs/using_helm/)
1)    add the repositories for **web-app**, **server**, and **infrastructure**
        ```shell
        helm repo add byor-voting-web-app https://raw.githubusercontent.com/thoughtworks/byor-voting-web-app/master/charts
        helm repo add byor-voting-server https://raw.githubusercontent.com/thoughtworks/byor-voting-server/master/charts
        helm repo add byor-voting-infrastructure https://raw.githubusercontent.com/thoughtworks/byor-voting-infrastructure/master/charts
        ```
1)    deploy BYOR-VotingApp:
        ```shell
        helm install byor-voting-chart
        ```

### Updating the application
To update the **VotingApp**, just repeat the step 3 above.


### HOWTOs

#### Access Kubernetes Dashboard:

Admin Username : k8s-admin
1)    get token
        ```shell
        kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep k8s-admin | awk '{print $1}')
        ```
1)    run the proxy
        ```shell
        kubectl proxy`` command in provision machine.
        ```
1)    access the dashboard at [http://localhost:8001/api/v1/namespaces/kube-system/services/](http://localhost:8001/api/v1/namespaces/kube-system/services/)


#### Validating certificate issuer.
```shell
kubectl describe clusterissuer <cluster issuer name>
kubectl -n istio-system describe certificate <certificate name>
```

### Access Kiali dashboard
```shell
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
```

### Access Jaeger dashboard
:warning: **[*TODO*]**  
```shell
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686  &
```

### Access Grafana dashboard
:warning: **[*TODO*]**  
```shell
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

### Generating certificates with Let's encrypt
:warning: **[*TODO*]**

### How to manage secrets
:warning: **[*TODO*]**  

## How to contribute to the project

Please refer to [CONTRIBUTING.md](CONTRIBUTING.md) for all the information about how to contribute.