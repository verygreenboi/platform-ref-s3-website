# platform-ref-s3-website

This repository defines a [Crossplane configuration package](https://docs.crossplane.io/latest/concepts/packages/#configuration-packages) that demonstrates provisioning and hosting static s3 websites.

## Composition Overview

The Amazon S3 static website hosting reference platform configuration serves as a foundation for building, running, and managing a static website using Amazon S3. A static website primarily consists of webpages that include fixed content and may incorporate client-side scripts.

```mermaid
flowchart LR

subgraph "XWebsite"
    CloudFront_Distribution["CloudFront \n Distribution"]
    CloudFront_Origin["CloudFront \n OriginAccessIdentity"]
    ACM_Certificate["ACM \n Certificate"]
    Route53_Record["Route53 \n A-Record"]
    Route53_Record2["Route53 \n CNAME"]
    S3_Bucket["S3 \n Bucket"]

    CloudFront_Distribution -->|Restricts Access| CloudFront_Origin
    CloudFront_Distribution --> Route53_Record
    ACM_Certificate --> Route53_Record2
    ACM_Certificate --> CloudFront_Distribution
    CloudFront_Distribution -->|Serves| S3_Bucket
end

subgraph "XContent"
    Object["S3 \n Object"]
    Object --> |Uploads| S3_Bucket
end

subgraph "XDNSZone"
    Zone["Route53 \n Zone"]
    Route53_Record --> Zone
    Route53_Record2 --> Zone
end
```

- [XWebsite](apis/XWebsite/): creates s3,cdn,acm,route53 configuration for static website hosting
  - [XContent](apis/XContent/): creates s3 object for static website hosting
  - [XDNSZone](apis/XDNSZone/): creates route53 public DNS Zone (use only if needed)

## Prerequisites

> **Note:** Before proceeding with the setup of the Amazon S3 static website hosting reference platform, please ensure that you have a working Route 53 Zone associated with your AWS Account. The Route 53 zone is essential to manage DNS records and route traffic to your website.
> If you already have a working Route 53 Zone with the domain you intend to use, you can proceed with the configuration of the static S3 website hosting. However, if you do not have a Route 53 zone set up for your domain, you must first create one. You may want to follow an example `example/zone.yaml` to ensure your zone is working correctly. 
> Once the Route 53 zone is in place, you can continue with the Amazon S3 static website hosting reference platform. 


First you will need access to a Kubernetes cluster. Ensure you are
using the correct context:

```sh
kubectl config current-context
```

Next, we'll use the `up` binary to install UXP, Upbound's distribution of
Crossplane. To get `up`, follow the [installation instructions](https://docs.upbound.io/uxp).

To install UXP using `up` run:

```console
up uxp install
UXP 1.12.1-up.1 installed
```

You can validate the install by inspecting all installed components:

```console
kubectl get all -n upbound-system
```
### Install the S3 Website Reference Platform

Now you can install this reference platform.

```console
kubectl apply -f examples/configuration.yaml
```

Validate the install by inspecting the provider and configuration packages:

```console
kubectl get providers,providerrevision

kubectl get configurations,configurationrevisions
```

Next, install the CompositeResourceDefinitions and Compositions:

```console
kubectl apply -f apis/XContent
kubectl apply -f apis/XWebsite
```

The Custom Platform APIs are Kubernetes `CompositeResourceDefinition` objects or `XRD`
for short. We can list them using `kubectl`:

```console
kubectl get xrd
```

The following XRDs should be `ESTABLISHED` and `OFFERED`:

```console
NAME                                       ESTABLISHED   OFFERED   AGE
xwebsites.example.upbound.io               True          True      109m
xcontents.example.upbound.io               True          True      109m
```

## Authenticating to AWS

Now that Crossplane, the Provider and all the Compositions are installed we
need to give the provider AWS credentials. This is done by creating a `ProviderConfig`.

There are many options we can use to authenticate to AWS, but to sim

```sh
kubectl create secret generic aws-creds -n upbound-system --from-file=creds=./creds.conf
```

## Authenticating with AWS

Once Crossplane, the Provider, and all the Compositions are installed, the next step is to provide AWS credentials to the Provider. This is achieved by creating a ProviderConfig.

For simplicity, we will use the following method to authenticate with AWS:

```sh
kubectl create secret generic aws-creds -n upbound-system --from-file=creds=./creds.conf
```

### Configuring the Provider with AWS Credentials

To configure the Provider with the AWS credentials obtained in the previous step, create the following `ProviderConfig` object. For more authentication options, such as IRSA, refer to  [AUTHENTICATION](https://github.com/upbound/provider-aws/blob/main/AUTHENTICATION.md).

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: aws-creds
      key: creds
```

Apply the configuration using the command:

```console
kubectl apply -f examples/providerconfig-creds.yaml
```

Now, the examples are ready to be deployed.

Using files in the `examples` directory:

```console
kubectl apply -f examples/ns.yaml
kubectl apply -f examples/website.yaml
kubectl apply -f examples/content.yaml
```

Applying the examples to the cluster would create Kubernetes objects similar
to the following:

```console
kubectl get websites
```

```console
NAME                                     SYNCED   READY   COMPOSITION                    AGE
xcontent.example.upbound.io/demo-9fsxm   True     True    xcontents.example.upbound.io   4m21s

NAME                                        DOMAIN                        ENABLED   SYNCED   READY   COMPOSITION                    AGE
xwebsite.example.upbound.io/website-vdc75   demo.website.upbound.io       true      True     True    xwebsites.example.upbound.io   37m
```

## Cleaning Up

To Clean up the installation, run the following commands:

```console
kubectl delete -f examples/content.yaml
kubectl delete -f examples/website.yaml
```

Wait for all the cloud resources to be deleted:

```console
kubectl get managed
```

Delete the Compositions, Providers, and ProviderConfig after all the resources have been deleted.

```console
kubectl delete -f examples/ns.yaml
kubectl delete -f apis/XWebsite
kubectl delete -f apis/XContent
kubectl delete -f examples/providerconfig-creds.yaml
kubectl delete -f examples/configuration.yaml
kubectl delete -f examples/provider-aws-scoped.yaml
```

## Local Development

This reference platform is a starting point to help you build your own
Platform APIs.

The following sections will detail how to make, test, and publish
modifications to these compositions.

### Setting Up the Build Environment

Clone this repository:

```console
git clone https://github.com/upbound/platform-ref-s3-website
```

Next pull in the Upbound [build](https://github.com/upbound/build) as a git submodule:

```console
cd platform-ref-s3-website
make submodules
Submodule 'build' (https://github.com/upbound/build) registered for path 'build'
Cloning into '/home/user/platform-ref-s3-website/build'...
Submodule path 'build': checked out '292f958d2d97f26b450723998f82f7fc1767920c'
```

Next run `make`. This will download the required components:

```sh
make
```

### Applying your Updated Compositions to a Cluster

### Automated Testing Using Uptest

[Uptest](https://github.com/upbound/uptest) is used for end to end testing of the
Compositions in this repository. It does this by provisioning example claims and
waiting for them to become `READY`.

To run uptest locally, first set the `UPTEST_CLOUD_CREDENTIALS` environment variable
with the contents of an AWS credentials file:

```sh
export UPTEST_CLOUD_CREDENTIALS=$(cat ~/.aws/credentials)
```

With the credentials file of the following format:

```ini
[default]
aws_access_key_id=AKIA...
aws_secret_access_key=jQplCPbh...
```

```sh
make e2e
```

## Questions?

For any questions, thoughts and comments don't hesitate to reach out or drop by slack.crossplane.io, and say hi!
