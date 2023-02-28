---
layout: post
title: "Debugging with ephemeral containers in K8s 🔲🔳"
date: "2020-05-30"
categories: 
  - "devops"
  - "kubernetes"
tags: 
  - "devops"
  - "kubernetes"
lang: en
lang-ref: ephemeral
---

![](/assets/img/imported/excelentes-utilidades-para-los-containers.jpg)

Have you ever faced a situation when working with a Kubernetes cluster in which a container within a pod is not working as expected and you need to do some debugging? I'm sure you have. The first step to follow in this troubleshooting scenario would be to try to figure out what's going on inside the container by checking the logs. But as you may have experienced, not all applications do provide useful logs, or any logs at all!

Well, you could always connect to the container by issuing _kubectl exec_ command and inspect everything from the inside, just to get a lead into what could be happening. But it could also happen that the image does not contain debug tools or a shell to run commands. So, now what?🤔 

It would be a great idea to have a 'sidecar' container running inside the same pod that we need to troubleshoot. Actually, it is a common practice, when the original image of the container cannot or shouldn't be modified to include additional utilities. However, this implies that the sidecar container would be running always within the pod, consuming resources whether if it's used or not.

Starting with version v1.18, Kubectl can create ephemeral containers and attach them to already running pods. This is ideal for debugging purposes in the situation described before.

```bash
kubectl alpha debug -it pod-name --image=busybox --target=pod-name
```

Under the hood, this command creates the ephemeral container and attaches it to the **process namespace** of the container we want to troubleshoot.

If you don't know what a process namespace is, you can go [through this article](https://medium.com/@teddyking/linux-namespaces-850489d3ccf). Long story short: a process running within a namespace 'sees' only what is in that namespace, preventing it from accessing resources or processes running in other containers (namespaces). Namespaces are a great part of the magic powering modern container runtimes🧙‍♂️.

Thus, by attaching the ephemeral container to the namespace, you can inspect the processes running within the container to troubleshoot.

For this example, the main container running in the pod is an nginx container. In the image below, you can check how the ephemeral container is attached to the pod and how this new container shares the process namespace of the nginx container.

![](/assets/img/imported/Screen-Shot-2020-05-30-at-2.22.39-AM-1024x376.png)

If you inspect the pod by issuing _kubectl describe pod_, you will notice that the new ephemeral container appears in the pod specifications. In this case, it is showing an error since the debugging session was closed after inspecting the container.

![](/assets/img/imported/Screen-Shot-2020-05-30-at-2.26.26-AM-1024x321.png)

* * *

## Enabling ephemeral containers

As you may have deduced by the command name, it is still an alpha feature, and it is not enabled in the cluster by default. To enable ephermeral containers, you need to set to true the feature gates flag of the Kubernetes control plane elements: API Server, Controller Manager, Scheduler and Kubelet. If your cluster was created using kubeadm, you can find the manifests for the three main elements in _/etc/kubernetes/manifests_ in the **master nodes**. Add the following line into the container command arguments.

```bash
- --feature-gates=EphemeralContainers=true
```

Take the _kube-apiserver.yaml_ as a reference, you should end up with something similar to the snippet below. Repeat this process for the Controller Manager and the Scheduler.

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - .....
    - .....
    - --feature-gates=EphemeralContainers=true
```

![](/assets/img/imported/Screen-Shot-2020-05-29-at-9.26.21-AM-1.png)

For the Kubelet, things are a little bit trickier, because it is the only component that cannot be containerized, since it is the element which runs the containers in each node. Kubelet is managed by means of systemd, so check the service configuration file _/etc/systemd/system/kubelet.service.d/10-kubeadm.conf_ in both the **worker** and **master** nodes. In the file, find the _EnvironmentFile_ variable, which tells you where the additional variables for the Kubelet are defined. In this case, it is _/var/lib/kubelet/kubeadm-flags.env_.

![](/assets/img/imported/Screen-Shot-2020-05-30-at-2.13.54-AM-1024x164.png)

Add the feature gates flag into the file _/var/lib/kubelet/kubeadm-flags.env_.

![](/assets/img/imported/Screen-Shot-2020-05-30-at-2.16.48-AM-1024x38.png)

Finally, restart the Kubelet service.

```bash
sudo service  kubelet restart
```

Now that everything has been set, you can check the flags by inspecting the processes running in the master and worker nodes.

![](/assets/img/imported/Screen-Shot-2020-05-29-at-9.27.38-AM-1024x95.png)

### **IMPORTANT**⚠️

In case your cluster was installed without kubeadm (a.k.a The Hard Way), the API Sever, Controller Manager and Scheduler are managed by means of systemd too. Thus, you will need to check the service configuration files in order to add the ephemeral containers flag.

* * *
