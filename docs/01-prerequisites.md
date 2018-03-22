# Prerequisites

## Amazon Web Services

This tutorial leverages the [Amazon Web Services](http://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html?type=resubscribe) for [free tier](https://aws.amazon.com/free/).

Estimated cost: TBD

> The compute resources required for this tutorial exceed the AWS free tier.

## AWS CLI

### Install the AWS CLI

Follow the AWS CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/installing.html) to install and configure the `aws` command line utility.

### Set a Default Compute Region

This tutorial assumes a default region and AWS credentials are configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure
```

Please refer to AWS CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) for more details.

Next: [Installing the Client Tools](02-client-tools.md)
