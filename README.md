# Infrastructure
<!-- MarkdownTOC -->

- [Summary](#summary)
- [Intro to Runway](#intro-to-runway)
    - [Terraform and Runway](#terraform-and-runway)
        - [Terraform layout](#terraform-layout)
    - [CloudFormation engin and Runway](#cloudformation-engin-and-runway)
- [Runway](#runway)
    - [Modules](#modules)
    - [tfstate.cfn](#tfstate)
    - [Prerequisites](#prerequisites)
        - [Python](#python)
            - [Windows](#windows)
            - [MacOS](#macos)
            - [Linux](#linux)
                - [Ubuntu](#ubuntu)
        - [AWS Cli](#aws-cli)
            - [Windows](#windows-1)
            - [MacOS](#macos-1)
            - [Linux](#linux-1)
                - [Ubuntu](#ubuntu-1)
        - [Runway and dependencies](#runway-and-dependencies)
        - [Terraform \(via TFEnv\)](#terraform-via-tfenv)
            - [MacOS](#macos-2)
            - [Linux](#linux-2)
        - [Other](#other)
    - [Deployments](#deployments)
        - [Interactive Deployments](#interactive-deployments)
        - [Non-Interactive Targeted Deployments](#non-interactive-targeted-deployments)
        - [Non-Interactive Deployments](#non-interactive-deployments)

<!-- /MarkdownTOC -->

## Summary
Infrastructure as Code using runway, CloudFormation and Terraform to launch into AWS.

## Intro to Runway
[Runway](https://docs.onica.com/projects/runway/en/latest/index.html) is configured for deployments and modules. A deployment defines modules and options that affect the modules.

Modules can deploy anything in one of the supported methodologies:
* [Terraform](https://www.terraform.io/docs/index.html)
* [Serverless Framework](https://www.serverless.com/framework/docs/)
* [CFNGin](https://docs.onica.com/projects/runway/en/latest/cfngin/configuration.html#cfngin-config-file)
* [CDK](https://docs.aws.amazon.com/cdk/api/latest/) or
* [Kubernetes](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

We separate the deployments into logical sections generally by use-case.

1. The first deployment option is [tfstate.cfn](#tfstate).
1. The second is designed around launch infrastructure type tooling, such as VPC's, Subnet's, Transit Gateway's.
1. The 3rd could be related to app deployments such as ECS or RDS
1. With the 4th being an app itself if that's being launched from this repo.

### Terraform and Runway

With the first deployment launching a S3 Bucket and DynamoDB Table this outputs from the Cloudformation the bucket and dynamodb table to be used as inputs to terraform deployments.

```terraform
  - name: Network
    modules:
      - path: core.tf
        name: Network-VPC-Subnets
        tags:
          - core
    parameters:
      namespace: ${env DEPLOY_ENVIRONMENT}
      region: ${env AWS_REGION}
    module_options:
      terraform_version: 0.15.1
      terraform_backend_config:
        region: us-east-2
        bucket: ${cfn ${env DEPLOY_ENVIRONMENT}-tf-state.TerraformStateBucketName}
        dynamodb_table: ${cfn ${env DEPLOY_ENVIRONMENT}-tf-state.TerraformStateTableName}
    regions:
      - us-east-2
    assume_role:  # optional
      devops: arn:aws:iam::${ACCOUNT_ID}:role/AWSControlTowerExecution
```
This is the Network deployment, pass in at the deployment level a list of modules, parameters, module_options, and regions.
* Parameters specify items to pass to the individual orchestration tool. In this case namespace and region are Terraform variables.
* Module Options specify configuration of the orchestration tool, this allows for configuration of the terraform backend and version at the deployment level. Notice though the key is omitted here as it is specified in `backend.tf`.
* Regions specifies what regions this deployment should be executed in.
* Assume Role this allows Runway to assume this role prior to executing in these accounts.

Modules are executed in the order specified.

This is the core.tf which launches the Network infrastructure such as VPC's and Subnet's. With this setup we have the relative path to the terraform directory `core.tf`, a pretty name (otherwise it would be referred to as just the path), and tags.

Tags allow a deployment to be specified directly on the cli (`runway plan --tag core`).

In the above code snippet if this were run as is it would attempt to execute for the `DEPLOY_ENVIRONMENT` of the role's listed in the region us-east-2 and launch `core.tf` in it. In all other accounts but `devops` it will prompt you for missing variables. As listed below there is a `devops-us-east-2.tfvars` which corresponds to both the assume role and the `DEPLOY_ENVIRONMENT`.

#### Terraform layout
In Terraform you can utilize multiple files to make editing easier and more readable. Generally the layout follows as below
```text
core.tf/
├── .terraform.lock.hcl
├── backend.tf
├── devops-us-east-2.tfvars
├── main.tf
├── outputs.tf
├── provider.tf
├── README.md
├── subnet
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
└── variables.tf
```

* `backend.tf` container configuration for state management.
* `main.tf` Sometimes the Terraform project is so small it doesn't make sense to break things out further so keeping everything else in `main.tf` makes sense.
* `outputs.tf` Has all of the outputs.
* `provider.tf` will list the required providers such as aws and their versions. Version pinning is important. The version will get locked in the `.terraform.lock.hcl` which if an update is made which changes that please commit the lock change to the repo.
* `README.md` It's customary to place a description of what this project does and the output from [Terraform Docs](https://terraform-docs.io/) (`terraform-docs markdown . > README.md`).
* modules belong in their own directory, if the modules is only used in one Terraform project then it should be in that projects directory, otherwise a folder in the root of the repo should be created for shared modules using the name `modules`.
* `variables.tf` holds all the variable declarations.
* `devops-us-east-2.tfvars` tfvars but naming pattern is determined by the region and deploy environment

### CloudFormation engin and Runway

CloudFormation engin(CFNGin) is a derirative of [Stacker](https://stacker.readthedocs.io/en/latest/) which expands on the functionality Stacker has but has diverged.

CFNGin allows you to deploy Cloudformation from JSON/YAML template files but also frol [Troposphere](https://troposphere.readthedocs.io/en/latest/) blueprints.

```
tfstate.cfn/
├── stacks.yaml
└── templates
    └── tf_state.yml
```

The overview is simpler in this use case compared to Terraform. Specifying all the variables in the runway config file avoids needing to use a `.env` similiar to a `.tfvars` file.

```yaml
  - name: Terraform State Bootstrap
    modules:
      - name: tfstate
        path: tfstate.cfn
        tags:
          - tfstate
    parameters:
      namespace: ${env DEPLOY_ENVIRONMENT}
      customer: ${REPLACE_ME}
      region: ${env AWS_REGION}
    regions:
      - us-east-2
```

From the runway snippet above namespace, customer, and region are passed in. In the below you see variables using these parameters in leiu of a environment file.

```yaml
namespace: ${namespace}
cfngin_bucket: ""  # not uploading any CFN templates

sys_path: ./

stacks:
  tf-state:
    template_path: templates/tf_state.yml
    variables:
      BucketName: ${customer}-${namespace}-${region}-terraform-state
      TableName: ${namespace}-${region}-terraform-lock
```

This is a fairly simple CFNGin deployment and omits usage of hooks and lookups.
This will launch a cloudformation stack with the name of the namespace and the stack shortname such as `devops-tf-state` using the template specified and passing in the variables.

## Runway
### Modules
### tfstate.cfn {#tfstate}

This is the only Cloudformation in this repo which bootstraps the state files for Terraform. It launches a s3 bucket and dynamodb table for state locking.

### Prerequisites
#### Python

##### Windows
1. Visit [Python](https://www.python.org/downloads/windows/) and download the latest version. 
1. Install by opening the exe, ensure that you install `pip` and add to path.
1. Open a terminal and run `pip install pipenv`

##### MacOS
1. Install [Brew](https://brew.sh/)
1. In a terminal `brew install python@3.9 pipenv`

##### Linux
Providing instructions for multiple distributions is out of scope.
###### Ubuntu
1. `sudo apt install python3-dev python3-pip pipenv`


#### AWS Cli
##### Windows
1. Follow instructions on https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html
##### MacOS
1. `brew install awscli`
##### Linux
###### Ubuntu
1. `sudo apt install awscli`

#### Runway and dependencies
1. navigate in your terminal to the directory of the repo.
1. `pipenv sync`
    1. [Pipenv](https://pipenv.pypa.io/en/latest/)

#### Terraform (via TFEnv)
##### MacOS
1. `brew install tfenv`
##### Linux
1. `sudo git clone https://github.com/tfutils/tfenv.git /usr/local/src/tfenv`
1. `sudo ln -s /usr/local/src/tfenv/bin/* /usr/local/bin`

#### Other
* This assumes an understanding of cloning a git repo.
* AWS account setup with MFA and API access, this account will need access equal to an administrator or the ability to assume a role with similar access

### Deployments
#### Interactive Deployments
1. login to your AWS account using CLI/MFA
1. Start runway with the correct deploy environment set with the following command:
    `DEPLOY_ENVIRONMENT=prod runway deploy`
1. Select your deployment.
```
1: deployment_1 - tfstate (us-east-2)
2: deployment_2 - Network-VPC-Subnets ()
3: deployment_3 - transit-gateway (us-east-2)
```
1. If there are multiple modules you will be prompted for which modules or all.
1. Deployment happens, watch for errors.

#### Non-Interactive Targeted Deployments
1. login to your AWS account using CLI/MFA
1. `runway deploy -e ${DEPLOY_ENVIRONMENT} --tag ${TAG}`

#### Non-Interactive Deployments
1. login to your AWS account using CLI/MFA
1. `runway deploy -e ${DEPLOY_ENVIRONMENT} --ci`
