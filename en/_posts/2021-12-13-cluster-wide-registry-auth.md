---
layout: post
title: "Cluster-wide registry authentication 🌐🔐"
date: "2021-12-13"
categories: 
  - "devops"
  - "docker"
  - "kubernetes"
tags: 
  - "docker"
  - "image"
  - "k8s"
  - "kubernetes"
lang: en
lang-ref: registry-auth
---

Using private container image registries is a common practice in the industry, as it ensures applications packaged within the images are just accessible for users which hold the right set of credentials. Besides, in some cases, using credentials for some registries helps overcoming pull rate limitations, for example when using [paid subscriptions for DockerHub](https://www.docker.com/pricing).

![](/assets/img/imported/461f0ea0da8f612dbea489d1bffe9d7d-1024x640.jpg)

Container ship by [Sketchfab](https://sketchfab.com)

The authentication process for image registries is pretty straightforward when using a container runtime locally, like Docker. In the end, all you need to do is setting your local configuration to use the right credentials. In the case of Docker, it can be done [running the login command](https://docs.docker.com/engine/reference/commandline/login/). However, when it comes to a Kubernetes cluster, in which each node runs its own container runtime, this process can become way more complex.

## Pulling private images 📦🔒

When we want to deploy images from private container registries in Kubernetes, first we need to provide the credentials so that the container runtime running in the node can actually authenticate before pulling them. The credentials are normally stored as a Kubernetes secret of type _docker-registry_, which stores the json-formatted authentication parameters in a base64 string.

{% highlight bash %}
$ kubectl create secret docker-registry mysecret \
  -n <your-namespace> \
  --docker-server=<your-registry-server> \
  --docker-username=<your-name> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
{% endhighlight %}

These credentials can be then used in two ways:

- Referencing the secret name with the _imagePullSecrets_ directive on the pod definition manifest, like in the snippet below. In this case the credentials will be just used for the pods that reference them.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: default
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: mysecret
```

- Attaching them to the default service account of a namespace. In this case, all the pods of the namespace will use the credentials when pulling the image from the private registry.

```
kubectl patch serviceaccount default \
  -p "{\"imagePullSecrets\": [{\"name\": \"mysecret\"}]}" \
  -n default
```

The second way of using the credentials, can provide cluster wide authentication, but in order to make it work, the cluster manager needs to create the secret in each namespace and patch the default service account of every namespace. Plus, if new namespaces are created, they are not automatically updated, so new secrets will need to be added and the service accounts patched.

Here is where the tool we will be reviewing comes in handy.

* * *

## Imagepullsecret-patcher 🤖🔒

[Imagepullsecret-patcher](https://github.com/titansoft-pte-ltd/imagepullsecret-patcher) is an open source project by [Titansoft](https://www.titansoft.com/en), a Singapore-based software development company. This solution eases setting cluster wide credentials to access image registries.

It is implemented as container image which contains a Kubernetes [client-go](https://github.com/kubernetes/client-go) application, which talks to the Kubernetes API to create an _imagePullSecret_ on each namespace and patches the default service account of the namespace so that it uses the secret. Furthermore, every time a new namespace is created, the process is automatically applied on it.

**NOTE:** There might be some special cases in which you may not want a namespace to use the cluster wide _imagePullSecrets_ (i.e. security reasons), but no worries, the patcher can be disabled for specific namespaces by adding a special annotation to the namespace.

```yaml
k8s.titansoft.com/imagepullsecret-patcher-exclude: true
```

* * *

## Deploying it ➡️☸️

First, let's create a specific namespace for the pullsecrets patcher. You can do it by issuing the following command.

```bash
kubectl create namespace imagepullsecret-patcher
```

Once the namespace is created, you need to create a secret containing your registry credentials. In order to do so, execute the following command. If you use Dockerhub as your registry, use _https://index.docker.io/v1/_ in the _\--docker-server_ field.

```bash
kubectl create secret docker-registry registry-credentials \
  -n imagepullsecret-patcher \
  --docker-server=<your-registry-server> \
  --docker-username=<your-name> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

Finally, deploy the rest of resources that the patcher needs to work. The following command will deploy a _[ClusterRole](https://v1-20.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)_ with permissions to create and modify secrets and as service accounts, as well as a service account for the patcher itself. They are mapped by means of a _[ClusterRoleBinding](https://v1-20.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding)_. Finally, it will create a deployment which mounts the secret you created manually and uses the service account.

```
kubectl apply -f https://raw.githubusercontent.com/mifonpe/pullsecrets-cluster-demo/main/manifests.yaml 
```

You can check all of the resources accessing [this file](https://github.com/mifonpe/pullsecrets-cluster-demo/blob/main/manifests.yaml) on the example repo.

Once the deployment is up, it's time to check if it worked. If you check the resources in the _imagepullsecret-patcher_ namespace, you should see something like this.

![](/assets/img/imported/Screenshot-2021-12-11-at-10.39.44-1024x250.png)

* * *

## Using it ⚙️☸️

Once the installation has succeeded, you can check the default namespace secrets. You should be able to see the secret created by the patcher.

**NOTE:** _k_ is an alias for _kubetcl_

![](/assets/img/imported/Screenshot-2021-12-11-at-10.40.08-1024x137.png)

Finally, if you check the default service account on the default namespace, it should have been patched in order to use the new secret.

![](/assets/img/imported/Screenshot-2021-12-11-at-10.40.47.png)

Let's create a new namespace to check how it works. Execute the following command to create a new, empty namespace.

```
kubectl create namespace test-patcher
```

Now let's execute the same commands that were executed in the default namespace. You will see how the secret was created by the image patcher and that the default service account was patched.

- ![](/assets/img/imported/Screenshot-2021-12-11-at-10.47.54-1024x452.png)
    

If you check the logs of the patcher container, you can see how it detected the newly created namespace and reacted to this event.

- ![](/assets/img/imported/Screenshot-2021-12-11-at-13.29.36-1024x88.png)
    

Finally, let's pull a private image. For this example I will be pulling a private image from Dockerhub that was constructed for [another post of the blog](https://kubesandclouds.com/index.php/2021/11/04/kaniko/).

![](/assets/img/imported/Screenshot-2021-12-11-at-13.22.54-1024x596.png)

In order to create a deployment which instantiates a pod using your private image, you can use the following command.

```
kubectl run test --image <your-private-image>:<your-tag>
```

Check if the pod is running.

- ![](/assets/img/imported/Screenshot-2021-12-11-at-13.40.50-1024x91.png)
    

If you describe the pod, you should be able to see how the private image was pulled by the cluster!

![](/assets/img/imported/Screenshot-2021-12-11-at-13.38.32-1024x154.png)

* * *

## How to make it better? ➕🤖

In my personal case, I manage most of the cluster add-ons and components using GitLab CI and [Helmfile](https://github.com/roboll/helmfile) (in case you never heard of it, you can find an article about Helmfile [here](https://kubesandclouds.com/index.php/2020/12/16/helmfile/)). My objective with this approach is to avoid any manual intervention on the cluster, so that bootstrap and disaster recovery procedures can be fully automated.

Thus, I install the patcher as a Helm release, and inject the variables from GitLab CI, avoiding manual secret creation or credentials leakage. Furthermore, I use Helm templating functions to inject the authentication string coming from the CI Variables. The snippet below shows how to implement this exact solution for the secret.

This solution can be extended to any CI/CD platform. The credentials are just configured once within your secrets storage or external secret vault and the CI will take care of deploying it into your cluster or set of clusters.

```
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: image-pull-secret-src
  namespace: imagepullsecret-patcher
data:
  .dockerconfigjson: {{ requiredEnv "DOCKERHUB_AUTH" }} 
```

* * *


