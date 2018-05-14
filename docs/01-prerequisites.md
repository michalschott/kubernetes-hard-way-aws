# Prerequisites

## AWS

This tutorial leverages the AWS to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up.

> The compute resources required for this tutorial exceed the AWS free tier.

### Install the AWS CLI

Follow the AWS CLI [documentation](https://aws.amazon.com/cli/) to install and configure the `aws` command line utility.

Verify the AWS CLI version is 1.15.10 or higher:

```
aws --version
```

### Configure AWS CLI

This tutorial assumes a default region have been configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure

AWS Access Key ID [None]: XXXXXXXXXXXXXXXXXXXX
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Default region name [None]: eu-west-1
Default output format [None]: json
```

Next: [Installing the Client Tools](02-client-tools.md)
