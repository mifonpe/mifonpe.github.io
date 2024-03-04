---
layout: post
title: "Detect API changes before k8s cluster upgrades ✅"
date: "2024-03-04"
tags: [kubernetes, EKS, containers]
lang: en
lang-ref: eksupgrade
--- 

Upgrading Kubernetes clusters is not always easy, even when you use managed Kubernetes platforms. One of the main issues we might encounter is API deprecations from one Kubernetes version to another. Normally, the API changes are announced and marked as deprecated for several versions before getting removed. But let's be honest, it is difficult to keep track of all resources and their API versions within a cluster, especially when multiple tenants deploy their workloads there.

Whenever you apply a Kubernetes manifest containing deprecated APIs, Kubernetes patches it on the flight so that it uses the new API version and configuration. Besides, it displays a message stating that the API is deprecated, like the one shown below. You can check the list of deprecated k8s APIs per version [here](https://kubernetes.io/docs/reference/using-api/deprecation-guide/).

![](/assets/img/deprecated.png)

But, what happens if you deploy an object with an API that is no longer supported? In this case, the API used by the object is not recognized and the object cannot be created or updated anymore.

![](/assets/img/failure.png)

Thus, upgrading the version of your cluster can have multiple unplanned and unforeseen consequences for the workloads and objects deployed if done blindly. 

![](/assets/img/k8s-fuckup.png)

In this post, we will review different tools that can be used to detect deprecated API versions in a Kubernetes cluster before undergoing an upgrade.


## EKS Upgrade Insights 🔍

In December 2023 AWS [launched](https://aws.amazon.com/about-aws/whats-new/2023/12/amazon-eks-upgrade-insights/) a new feature within EKS called Upgrade Insights, which allows you to identify deprecated APIs within the EKS console.

First, you need to select the cluster you want to check and click on the Upgrade Insights tab.

![](/assets/img/api1.png)

Then you should be able to see the warnings for the different versions available.

![](/assets/img/api2.png)

Finally, for each version, you can see all the deprecated APIs and the proposed replacements!

![](/assets/img/api3.png)

## CLI tools 💻

You can also use CLI tools to detect deprecated APIs in a more programmatic way.

### AWS CLI 🟠

Upgrade insights can be executed directly from the AWS CLI. It can be executed as follows and it will return a list of upgrade insights if applicable.

```
aws eks list-insights --filter kubernetesVersions=1.25 --cluster-name preflight | jq .
```
Take note of the insight you want to analyze, as we will use its ID in the next command.

![](/assets/img/insight.png)

Using the following command you can see the details of each particular insight.

```
aws eks describe-insight --id <your-insight-id> --cluster-name preflight | jq .
```

![](/assets/img/insight2.png)

### Pluto 🐶

This is a simple CLI utility developed by [Fairwinds](https://www.fairwinds.com/) which is the one I have been using for the last couple of years, as the upgrade insights are relatively new. You can access its GitHub repository [here](https://github.com/FairwindsOps/pluto).

Pluto can find deprecated resources within Helm releases.

![](/assets/img/pluto-1.png)

However, I prefer to apply it to the whole cluster, as there might be some resources that have not been deployed with helm (*detect-all-in-cluster*).

![](/assets/img/pluto-2.png)

Finally, pluto can be also used to analyze Kubernetes manifest files, making it useful as a tool that can be integrated within your CI/CD flows.

```
pluto detect-files -d <your-directory>
```

### Other tools 🧰

Some other interesting tools offer similar capabilities. Feel free to look at these repositories.

- [Kubedd](https://github.com/devtron-labs/silver-surfer?tab=readme-ov-file) --> follows a similar approach to Pluto
- [Kubeconform](https://github.com/yannh/kubeconform) --> a manifest validation tool
- [Kubent](https://github.com/doitintl/kube-no-trouble) --> similar to Pluto as well
- [Kubepug](https://github.com/kubepug/kubepug) --> kubectl plugin

## Extra: Patch deprecated Helm releases ☸️

As commented, when you apply a deprecated API, k8s automatically translates the object to the new API definition. However, when using Helm, it is possible to reach a situation in which the objects in the cluster have the right API version (patched by k8s) but the release data stored in the cluster has an older, incompatible version. This will affect the release even if you modify the code of your Helm templates!

When this happens, you can see errors like this one:

```
Error: UPGRADE FAILED: current release manifest contains removed kubernetes api(s) for this kubernetes version and it is therefore unable to build the kubernetes objects for performing the diff. 

error from kubernetes: unable to recognize “”: no matches for kind “Ingress” in version “networking.k8s.io/v1beta1”
```

In this case, you can follow [this documentation](https://helm.sh/docs/topics/kubernetes_apis/#updating-api-versions-of-a-release-manifest) from Helm to patch the broken release. All it does is decode and decompress the release data stored as a secret, patch the content, compress it and encode it again.

```
kubectl get secret <release_secret_name> -n <release_namespace> -o yaml > release.yaml
cp release.yaml release.bak # back up the release just in case
cat release.yaml | grep -oP '(?<=release: ).*' | base64 -d | base64 -d | gzip -d > release.data.decoded
```

By now you should have a plaintext file containing all the data of your release (release.data.decoded), find your deprecated APIs and patch them.

Finally, compress and base64 encode the contents again and modify the contents of release.yaml with this value.

```
cat release.data.decoded | gzip | base64 | base64
```

Once this is done, all that is left is to apply the file to the cluster!

```
kubectl apply -f release.yaml -n <release_namespace>
```
