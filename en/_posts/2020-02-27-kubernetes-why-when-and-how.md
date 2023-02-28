---
layout: post
title: "Kubernetes: Why, When and How."
date: "2020-02-27"
categories: 
  - "kubernetes"
tags: 
  - "containers"
  - "kubernetes"
  - "microservices"
lang: en
lang-ref: k8s-why
---

Kubernetes, is undoubtedly one the most disruptive tools/technologies that we have seen during the last years within the IT world. Alongside containers and the underlying runtimes, it is redefining the way in which software is built, tested and deployed into production. Although it is more complex to set up and to operate than some other container orchestration tools, it is still the most used solution for container orchestration.

![](/assets/img/imported/57m-Yacht-FLEURTJE-7929-82-300x200.jpg)

## Why kubernetes?

Following the microservices paradigm, services can be packed and shipped in small containers with all the "stuff" they need to run, guaranteeing immutability, isolation between processes and a decoupled architecture, which makes upgrading individual services easier and resource usage more efficient. But containers are just one part of the equation, orchestration is needed to achieve microservices' architecture maximum potential, providing fault tolerance, scheduling and communication between the different pieces. Here is where container orchestration starts to make sense, and specially Kubernetes.

Kubernetes comes in particularly handy when it comes to scalability and reliability. It uses auto scaling according to the demand, both within the cluster (scaling the internal objects) and for the cluster machines. One of its main functions is to ensure that the desired number of replicas of each component is running as expected. If any of them is not behaving as it should, it will roll out new components and kill the faulty ones. At the end of the day, they are just containers!

Due to its popularity, there is an enormous community of users, which have already solved many of the problems you might find while starting with Kubernetes. Plus, Kubernetes documentation is pretty useful and detailed, helping to reduce the steepness of Kubernetes learning curve. Furthermore, there are quite a lot of tools and initiatives to make Kubernetes somehow "user friendly" as well as to make it suitable for different environments, where flexibility, scalability and tailoring is a must. Some of the most important tools are those that add templating features to Kubernetes podspecs (the yaml files which contain Kubernetes objects definitions): helm and kustomize. Helm is specially useful since it provides "versioning" (kinda) for the deployments as well as rolling updates, minimizing application downtime.

Kubernetes clusters can be integrated with most of the modern CI/CD software, which makes it a perfect match for a seamlessly integrated DevOps environment. GitLab, GitHub, Bitbucket and Jenkins, amongst some others, can run their pipelines taking advantage of Kubernetes capabilities.

Kubernetes itself is highly customizable, which in turn makes it highly flexible (and sometimes highly difficult to manage 😅). Almost every single piece of it can be tweaked to meet the requirements or needs of each environment or project, so if you don't like something in Kubernetes, just do it your way. However, if you are afraid of managing a complete cluster, most of cloud providers offer managed solutions, hiding part of the complexity, and helping organizations to focus on the development of services rather than on maintenance issues.

But, what about the other orchestrators in the market? Well, while docker swarm is tightly integrated with the docker daemon and the docker API, its scalability and flexibility cannot compare to the one offered by Kubernetes, which is more suited for production environments in a medium to large scale. Nevertheless, if a larger (global) scale is needed, Kubernetes might not be an adequate solution even though it can be federated in a multi-cluster configuration, [Apache Mesos](http://mesos.apache.org/) is a better solution for those scenarios.

## When?

Well, obviously, if your company is following a modern microservices architecture (forget about Netflix's🤣), Kubernetes is the way to go. It allows developers to ship microservices within containers which can be versioned, making upgrades and rollbacks to previous versions an easier task. But keep in mind that Kubernetes is not magic, there is a need for a team and organization culture which fosters the DevOps way of thinking. The development team must be open to learn and experiment with new and complex technology. It takes some time to fully understand its architecture and how to work with it.

Kubernetes can be an useful tool if you want to reduce the time to the market for your software products, as it empowers development teams to build and test their code faster, by leveraging container technologies. It takes care of the availability, scalability and visibility. Kubernetes implements rolling updates, which helps reducing application downtime while performing upgrades. Besides it also supports canary deployments. That is what makes it a perfect ecosystem for DevOps best practices.

Portability is probably one of the main advantages of Kubernetes. Once defined, Kubernetes objects can be run in any cluster, no matter the infrastructure (however the cluster version is the limiting factor here). Kubernetes abstracts the underlying resources, eliminating dependencies with the OSs, presenting these resources as pools to the elements which are running in the cluster. Thus, it can lead to a better resource usage. Therefore, if you focus on the development of your services rather than on the hardware where it will be run, Kubernetes is a good option. Build your components, test them and deploy them into different environments, scaling them up as it is needed to meet the demand.

Lastly, remember that not everything is suitable be to be containerized and orchestrated as microservices. Some monolithic applications need to be completely reengineered so that they can be run in Kubernetes, and some other can't just be rearchitected. They are simply not cloud-native! Furthermore, there are some applications that can be run in Kubernetes but that are not suitable for real production environments, or for a highly intensive use. Its performance can be rather poor compared to the one provided by the same applications when they are deployed in VMs using HA deployments.

**My advise: Don't overcontainerize! Don't create "macroservices" by packing monolithic monsters into containers!**

## How?

There are two main ways of running Kubernetes clusters: the old hard way, either on-prem or cloud-based, and managed Kubernetes services. Which one is the best? Well, that is not an easy question, and the answer depends heavily on your needs and requirements regarding flexibility, customization and management overhead.

Installing Kubernetes from scratch is not an easy task, even though it has become easier as new versions were released 😅.

![](/assets/img/imported/EQZaQh1XUAEdDBP.jpeg)

Thus, there are several tools that can be pretty useful in order to deploy and manage a self hosted Kubernetes cluster:

- [Kops:](https://github.com/kubernetes/kops) a cli-like tool deploy and manage clusters into AWS, GCP and VMware. It can generate terraform and cloudformation outputs.
- [Kubespray:](https://github.com/kubernetes-sigs/kubespray) a highly customizable set of ansible roles to deploy Kubernetes clusters both on public cloud providers and on-premise.

When Kubernetes is deployed as a self hosted service, you need to take care of both the master nodes, which run the control plane, and the worker nodes, which run the containers and their workloads. This implies a lot of work to keep everything running as it should, making upgrades a rather complex process. Disaster recovery for the control plane has to be implemented as well. On the other hand, being able to configure the control plane provides greater flexibility, such as configuring the API server parameters or modifying the scheduler behaviour, but as you all know, _with great power comes great responsibility_ 🕷️.

If you don't really need to tune the cluster performance, and your main focus is to develop and deliver software in a fast and efficient way without worrying about the control plane, just choose managed services. These services, which are offered by all the main cloud providers, abstract the entire control plane, leaving just the worker nodes as elements to manage and to interact with.

Besides, there are hybrid solutions, such as [Rancher](https://rancher.com/products/rancher/), which can be used to deploy clusters in bare-metal, self hosted environments and using managed services. However, this tool is intended for organizations which need to manage a high number of clusters in a platform agnostic way.

* * *
