---
title: "Ephemeral Kubernetes Lab with IaC and GitOps"
date: 2021-12-08T09:05:42+11:00
draft: false
tags: ['aws', 'kubernetes', 'eks', 'terraform', 'iac', 'argocd', 'gitops']
categories: ['tech']
---

I've been thinking of moving my Kubernetes lab into the cloud, but with cloud resource usage being scrutinized by the IT department, running them 24x7 the way I'm used to is a no-go. I need a setup that meets the following requirements:

- Simple to create and tear down
- Applications must be pre-deployed when the cluster is up, as close to "just the way I left it there last night" as possible
- cost $0 when the setup has been switched off

I eventually settled on the idea of an ephemeral Kubernetes lab environment using [Infrastructure as Code (IaC)](#infrastructure-as-code-using-terraform) and [GitOps](#k8s-resource-management-via-gitops-using-argo-cd) practices, which I will cover in this post. You can follow along by cross referencing the code found here:

> [https://github.com/leonseng/terraform-everything/tree/master/eks-gitops](https://github.com/leonseng/terraform-everything/tree/master/eks-gitops)

# Infrastructure as Code using Terraform

Starting with the Kubernetes cluster, using a managed Kubernetes offering makes sense for me as my current focus is on Kubernetes applications.

> If cluster customization is of importance, one can deploy Kubernetes on the cloud computes using tools like [kops](https://github.com/kubernetes/kops) or [kubespray](https://github.com/kubernetes-sigs/kubespray), similarly adopting IaC practices detailed below.

I went with [EKS](https://aws.amazon.com/eks/) as I am more familiar with AWS. An EKS cluster (or any other managed Kubernetes offerings in the public cloud) has a lot of dependencies on other cloud resources, such as computes, gateways, security policies and more. Rather than figuring out *when* to create *what*, [Terraform](https://www.terraform.io/) can be used to define all the cloud resources needed for a functional EKS cluster in a declarative manner. In addition, Terraform modules such as [vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) and [eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) can be used to further abstract away the web of dependencies. All it takes is a couple of Terraform modules and some data sources to create the cluster, as seen here:

> [https://github.com/leonseng/terraform-everything/blob/master/eks-gitops/eks.tf](https://github.com/leonseng/terraform-everything/blob/master/eks-gitops/eks.tf)

With the cluster sorted, I turn my attention to deploying Kubernetes resources/applications in the cluster.

# K8s resource management via GitOps using Argo CD

It's instinctive to start running `kubectl` commands to deploy Kubernetes resources that make up your application in the cluster, but keeping track of them gets harder overtime. If the cluster is to be rebuilt on a regular basis, we'd best hope we have a record of what's deployed in it, and what better option than storing the Kubernetes manifests in a Git repository where the IaC definitions are also stored. With the desired state of applications in Git, the next step is ensuring the cluster state reflects the desired state. This is where GitOps comes in.

GitOps can be briefly described as an automated workflow that ensures the state of the cluster and its applications matches the desired state from the source of truth, or Git repositories in the case. GitOps achieves this with one of the two patterns below:

1. Changes are **pushed** to the cluster

    Continuous deployment workflow applies the resource manifests on our cluster whenever there are changes in the Git repository. This can be done using simple Bash scripts, or orchestration tools like Ansible. A problem arises when there are multiple deployment workflows targeting the same cluster. As each workflow may not have a complete view of the cluster, it could lead to an inconsistent state in the cluster.

1. Changes are **pulled** into the cluster

    Officially the preferred pattern in [OpenGitOps v1.0.0](https://opengitops.dev/blog/1.0-announcement/) - an agent in the cluster continuously polls the desired state from a remote Git repository, and applies the changes in the cluster. [Argo CD](https://argo-cd.readthedocs.io/en/stable/]) and [Flux](https://fluxcd.io/docs/) are two popular GitOps tools that employs this pattern.

For this article, I will be using Argo CD, mainly for its explicit support for the [App of Apps pattern](#the-one-app-to-rule-them-all) which I will cover in a section further down.

## Argo CD Applications

In Argo CD, a Kubernetes application are defined via an [`Application` custom resource (CR)](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications). An `Application` CR provides Argo CD with the following information:

- **source**: the Git repository, revision and path to where the collection of Kubernetes resource manifests are stored
- **destination**: the target cluster (Argo CD supports multi-cluster GitOps) and namespace to apply/deploy Kubernetes resource manifests in

An `Application` CR can be as simple as this:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

The diagram below shows how an application is deployed with Argo CD:

![Basic Argo CD workflow](/images/argocd-basic.png)

1. User creates/modifies and uploads Kubernetes resource manifests (e.g. Deployment, ConfigMap, Service etc) for an application into a Git repository.
1. (**manual step in cluster**) User deploys an `Application` CR.
1. Argo CD inspects `Application` CR to discover the Git repository.
1. Argo CD pulls the Kubernetes resource manifests from the Git repository.
1. Argo CD applies/deploys the Kubernetes resource manifests into the target cluster and namespace per the `Application` CR.

And just like that, our application deployment has been turned into code. Pretty cool! But we can do better...

## The One App To Rule Them All

The previous workflow still involves a manual step on the cluster - creating the `Application` CR in the cluster. If the user forgets or makes a mistake, the state of the cluster will not reflect the desired state defined in the Git repository. In order to deal with such scenarios, let's turn to the [App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern).

Remember that an `Application` CR just points the agent to where the Kubernetes resource manifests are, there's no limitation on what kind of resources it supports. Naturally, this can be extended to other `Application` CRs! Here's how it works:

![Argo CD app of apps workflow](/images/argocd-app-of-apps.png)

Let's start with the <span style="color:red">**red**</span> flow that shows the bootstrapping:
1. User creates a bootstrap Git repository to store `Application` CRs for the actual applications we want to deploy.
1. User deploys a bootstrap `Application` CR on the cluster.
1. Argo CD inspects the bootstrap `Application` CR to discover the bootstrap Git repository.
1. Argo CD agent starts monitoring the Git repository for new `Application` CRs.

Once the cluster has been bootstrapped (which only has to be done once when the cluster is being set up), let's go through the <span style="color:green">**green**</span> flow to see how applications are deployed/modified:
1. User creates/modifies and uploads Kubernetes resource manifests (e.g. Deployment, ConfigMap, Service etc) for an application into an application Git repository.
1. User creates/modifies and uploads an `Application` CR for the above application into the bootstrap Git repository.
1. Argo CD discovers new `Application` CR in the bootstrap Git repository.
1. Argo CD deploys the `Application` CR in the cluster, and from it, discover the application Git repository.
1. Argo CD pulls the Kubernetes resource manifests from the application Git repository
1. Argo CD applies the Kubernetes resource manifests into the target cluster and namespace per the `Application` CR.

Using the App of Apps pattern means we only have to maintain the desired state of our applications through Git, and Argo CD would automatically apply the desired state in the cluster.

Phew... that was a lot to take in 😅. Hopefully we are now in the right head space to move on to the next section where we automate the bootstrapping via Terraform.

## Terraforming GitOps

To integrate this GitOps workflow and the App of Apps pattern with the [IaC](#infrastructure-as-code-with-terraform) definitions, we use Terraform to:

1. Install Argo CD in the cluster

    I found two Terraform providers which can manage Kubernetes resources:
    - [kubernetes](https://registry.terraform.io/providers/hashicorp/kubernetes/latest) - officially supported by HashiCorp, but requires Kubernetes resources to be defined in the Terraform's syntax HCL instead of YAML. It also does not support defining a CRD and any corresponding CRs within the same Terraform project, as the CRD won't exist when Terraform is processing the CRs in the planning stage.
    - [kubectl](https://registry.terraform.io/providers/gavinbunney/kubectl/latest) - allows you to work in YAML and supports overwriting namespaces in the manifest. However, when performing `terraform destroy`, it does not seem to wait for resources to be fully deleted before marking them as destroyed (see [issue](https://github.com/gavinbunney/terraform-provider-kubectl/issues/109)).

    Neither of them are perfect, but I'm inclined to use the [kubernetes](https://registry.terraform.io/providers/hashicorp/kubernetes/latest) provider for the official support, and deviate when it doesn't work.

1. Bootstrap the cluster with an `Application` CR to utilize the App of Apps pattern.

    As explained earlier, this `Application` CR (see [manifest](https://github.com/leonseng/terraform-everything/blob/master/eks-gitops/bootstrap-app.yaml.tpl)) is responsible for defining the deployment of all other applications that we actually want running in our cluster. It simply points to a repository that looks something like this:

    > [https://github.com/leonseng/terraform-everything/tree/master/eks-gitops/demo/argocd-apps](https://github.com/leonseng/terraform-everything/tree/master/eks-gitops/demo/argocd-apps)

    where there are two applications to be deployed - [httpbin](https://github.com/leonseng/terraform-everything/tree/master/eks-gitops/demo/httpbin) and [nginx](https://github.com/leonseng/terraform-everything/tree/master/eks-gitops/demo/nginx).


The full Terraform file can be seen here:

> [https://github.com/leonseng/terraform-everything/blob/master/eks-gitops/argocd.tf](https://github.com/leonseng/terraform-everything/blob/master/eks-gitops/argocd.tf)

# Finale

With the Terraform files and application manifests in their respective Git repositories, we can kick off the deployment with the Terraform commands
```
terraform init
terraform apply -auto-approve
```

And at the end of the day, shutting it down is as easy as running
```
terraform destroy -auto-approve
```

Is it perfect? No. I need to be conscious of storing my application manifests in Git, which will slow me down. And occasionally, the destroy process fails due to some orphaned cloud resources (work in progress). But what I have now is an on/off button for my Kubernetes lab in the cloud, which should keep the IT department of my back 😁.
