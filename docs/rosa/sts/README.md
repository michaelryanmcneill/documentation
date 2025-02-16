# Creating a ROSA cluster in STS mode

**Paul Czarkowski**

*Last updated 10/29/2021*

> **Tip** The official documentation for installing a ROSA cluster in STS mode can be found [here](https://docs.openshift.com/rosa/rosa_getting_started_sts/rosa-sts-getting-started-workflow.html).


STS allows us to deploy ROSA without needing a ROSA admin account, instead it uses roles and policies with Amazon STS (secure token service) to gain access to the AWS resources needed to install and operate the cluster.

This is a summary of the [official docs](https://docs.openshift.com/rosa/rosa_getting_started/rosa-sts-getting-started-workflow.html) that can be used as a line by line install guide and later used as a basis for automation in your [favorite automation tool](https://github.com/ansible/ansible).

> Note that some commands (OIDC for STS) will be hard coded to US-EAST-1, do not be tempted to change these to use $REGION instead or you will fail installation.

## Prerequisites

* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
* [Rosa CLI](https://github.com/openshift/rosa/releases/tag/v1.1.5) v1.1.5
* OpenShift CLI - `rosa download openshift-client`
* [jq](https://stedolan.github.io/jq/download/)

### Prepare local environment

1. set some environment variables

    ```bash
    export VERSION=4.9.0 \
           ROSA_CLUSTER_NAME=mycluster \
           AWS_ACCOUNT_ID=`aws sts get-caller-identity --query Account --output text` \
           REGION=us-east-2 \
           AWS_PAGER=""
    ```

### Prepare AWS and Red Hat accounts

1. If this is your first time deploying ROSA you need to do some preparation as described [here](../../quickstart-rosa.md#Prerequisites). Stop just before running `rosa init` we don't need to do that for STS mode.


1. If this is a brand new AWS account that has never had a AWS Load Balancer installed in it, you should run the following

    ```bash
    aws iam create-service-linked-role --aws-service-name \
    "elasticloadbalancing.amazonaws.com"
    ```

## Deploy ROSA cluster

1. Make you your ROSA CLI version is correct (v1.1.5 or higher)

    ```bash
    rosa version
    ```

1. Create the IAM Account Roles

    ```
    rosa create account-roles --mode auto -y
    ```

1. Run the rosa cli to create your cluster

    > You can run the command as provided in the ouput of the previous step to deploy in interactive mode.

    > Add any other arguments to this command to suit your cluster. for example `--private-link` and `--subnet-ids=subnet-12345678,subnet-87654321`.

    ```bash
    rosa create cluster --sts --cluster-name ${ROSA_CLUSTER_NAME} \
      --region ${REGION} --version ${VERSION}
    ```

1. Create the Operator Roles

    ```bash
    rosa create operator-roles -c $ROSA_CLUSTER_NAME --mode auto --yes
    ```

1. Create the OIDC provider.

    ```bash
    rosa create oidc-provider -c $ROSA_CLUSTER_NAME --mode auto --yes
    ```

1. Validate The cluster is now installing

    The State should have moved beyond `pending` and show `installing` or `ready`.

    ```bash
    watch "rosa describe cluster -c $ROSA_CLUSTER_NAME"
    ```

1. Watch the install logs

    ```bash
    rosa logs install -c $ROSA_CLUSTER_NAME --watch --tail 10
    ```

## Validate the cluster

Once the cluster has finished installing we can validate we can access it

1. Create an Admin user

    ```bash
    rosa create admin -c $ROSA_CLUSTER_NAME
    ```

1. Wait a few moments and run the `oc login` command it provides.

## Cleanup

1. Delete the ROSA cluster

    ```bash
    rosa delete cluster -c $ROSA_CLUSTER_NAME
    ```
1. Clean up the STS roles

Once the cluster is deleted we can delete the STS roles.

    > Note you can get the correct commands with the ID filled in from the output of the previous step.

    ```bash
    rosa delete operator-roles -c <id> --yes --mode auto
    rosa delete oidc-provider -c <id>  --yes --mode auto
    ```
