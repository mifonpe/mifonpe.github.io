---
title: "Werf: Fully customizable GitOps 🛥️⚙️"
date: "2020-09-01"
categories: 
  - "gitops"
  - "kubernetes"
tags: 
  - "devops"
  - "gitops"
  - "kubernetes"
lang: en
lang-ref: werf
---

This is the third post of a collection of GitOps tools articles. In this post, a continuous integration and continuous deployment tool for Kubernetes is reviewed: **Werf**. If you missed previous articles on GitOps tools, you could give them a read, as it will help you better understanding the general idea. [ArgoCD](https://kubesandclouds.com/index.php/2020/05/13/gitops/) and [FluxCD](https://kubesandclouds.com/index.php/2020/05/24/gitops-fluxcd/), were the previously reviewed tools.

As for the previous tools, an example repository has been developed so that you can test Werf with some existing configurations. You can find the repository [here](https://github.com/mifonpe/werf-kubesandclouds).

![](images/Shipyard-1024x717.png)

Shipyard by [vecteezy](https://www.vecteezy.com/)

## GitOps & Werf ⛵

GitOps is defined as a way of managing Kubernetes infrastructure and applications, in which git repositories hold the declarative definition of what has to be running in Kubernetes. A git repository is considered as the _single source of truth_ in the GitOps model. With this approach, changes in the application definitions (code), are reflected on the infrastructure. Besides, it is possible to detect differences and drifts between the source controlled application and the different versions deployed.

GitOps allows faster delivery rates, as once the code is pushed to the production branch, it will be present in the cluster in a matter of seconds, increasing productivity. This Model also increases reliability, as rollbacks and recovery procedures are eased by the fact that the entire set of applications is control versioned by Git. Furthermore, stability and consistency is also improved.

[Werf](https://werf.io/) is an open source GitOps tool which aims to speed up application delivery by automating and simplifying the complete application lifecycle. To do so, Werf builds and publishes images, deploys applications to Kubernetes clusters, and removes unused images based on policies and rules defined in the Git repository.

![](images/werf-schema.png)

Werf flow by [werf.io](https://werf.io/)

Werf can build Docker images from Dockerfiles or using and alternative builder, which uses a custom syntax, and can support Ansible as well as incremental rebuilds based on Git history.

Werf deploys applications to Kubernetes using Helm-compatible charts with more extensive customizations. Furthermore, it provides rollout tracking, error detection and logging mechanisms for the application releases. The logic driving these mechanisms can be customized by means of Kubernetes annotations.

Werf is distributed as CLI written in go, so it can run virtually in any modern OS and it can be easily integrated with any CI/CD platform, as it has been developed to do so.

* * *

## Installing Werf 💻

In order to install Werf CLI, first you need to install Docker and Git in your machine. In case you haven't installed them yet, you can find the installation instructions for Docker [here](https://docs.docker.com/get-docker/), and for Git [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git). Once these dependencies are present, it's time to install Werf.

Werf advises to install their CLI by means of the [multiwerf](https://werf.io/documentation/guides/installation.html#method-1-recommended-by-using-multiwerf) utility, which ensures that the Werf CLI is always up to date. Execute the following commands to get it installed in your machine.

```bash
export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
mkdir -p ~/bin
cd ~/bin
curl -L https://raw.githubusercontent.com/werf/multiwerf/master/get.sh | bash
```

Select the Werf version to use and configure your terminal session to use that version by issuing the following command. Werf provides [5 stability levels](https://werf.io/releases.html) to choose from, so that you can choose a level depending on your environments and needs.

```bash
. $(multiwerf use 1.1 stable --as-file)
```

In case you don't want to use multiwerf features, you can just follow the 'old good' approach and get a specific version of the Werf CLI binary form [here](https://werf.io/documentation/guides/installation.html#method-2-by-downloading-binary-package).

* * *

## Project structure 📁

Werf expects files to be organized following a specific structure within the code repository. The image below shows the structure of the example repository that we will be using for this post, which contains a simple webpage (in **/web**) that is packaged in a nginx Docker image, and deployed to a Kubernetes cluster using Helm.

![](images/Screen-Shot-2020-08-31-at-9.47.02-AM.png)

On the top level of the repository you can find **werf.yaml**, a YAML configuration file for Werf which defines how to build the images, and contains some metadata and directives that allow customizing the behavior of Werf. This file contains metadata information, images build information and artifacts build information each part separated by three hyphens. Werf also supports Go templates for the **werf.yaml** file, as you can notice in the last line of the following example, making configurations really powerful. Besides, by including different templates, **werf.yaml** can be split into different files, to better handle complex configuration scenarios.

```yaml
project: my-project
configVersion: 1
---
image: app
from: java:8-jdk-alpine
...
---
{{ include "artifact/appserver.tmpl" . }}
```

At the same level you can find a **Dockerfile**, used to build the image. This **Dockerfile** can be directly referenced from **werf.yaml** to be used. However, Werf supports custom syntax to build images, directly specified within **werf.yaml** as you will see in the upcoming examples.

**.helm** directory contains the Helm charts to be used by Werf to deploy the application, organized following the [Helm chart file structure](https://helm.sh/docs/topics/charts/#the-chart-file-structure). Notice that **Chart.yaml** is not included as it is not used by Werf.

* * *

## Building images 📦

First, clone the examples repository to get started.

```bash
git clone https://github.com/mifonpe/werf-kubesandclouds.git
cd werf-kubesandclouds
```

Building images from a Dockerfile is the easiest way to start using Werf in an already existing project. To do so, you will only need to add the _dockerfile_ directive with the Dockerfile name (Dockerfile in this case) under the image config in your **werf.yaml** file.

```yaml
project: kyc-project
configVersion: 1
---
image: web-server
dockerfile: Dockerfile
```

The code below shows the contents of the Dockerfile. It is fairly simple, as it just adds the webpage and its media to the image and exposes the port 80 fort the nginx server to listen on.

```bash
FROM nginx:latest

COPY web/index.html /usr/share/nginx/html
COPY web/kyc.png /usr/share/nginx/html
EXPOSE 80 
CMD ["nginx", "-g", "daemon off;"]
```

However, Werf provides advanced features for image building, eliminating the need for Dockerfiles, although they can be combined with the Werf builder syntax. Besides, you can build as many different images as you need using just one Werf configuration file ([multi-image build](https://werf.io/documentation/guides/advanced_build/multi_images.html)). The code below shows the contents of the **werf.yaml** file of the [example repository](https://github.com/mifonpe/werf-kubesandclouds).

```yaml
project: kyc-project
configVersion: 1
deploy:
  helmRelease: werf-test
  helmReleaseSlug: false
  namespace: default
---
#image: web-server
#dockerfile: Dockerfile

image: werf-web
from: nginx:latest

ansible:
  install:
  - debug:
      msg: 'Start install'
  - file:
      path: /usr/share/nginx/html
      mode: 0755
  - file:
      path: /web
      state: directory
      mode: 0755
  - debug:
      msg: 'End install'
      
git:
- add: /web
  to: /usr/share/nginx/html
  owner: nginx
      
#shell:
#  beforeInstall:
#  - mkdir /web
  
docker:
  WORKDIR: /web
  CMD: ["nginx", "-g", "daemon off;"]
  EXPOSE: '80'
  ENV:
    BUILDER: werf-custom 
```

In this case, the image is built using different Werf directives. First, Ansible syntax is used with the _ansible_ directive, to modify and create some directories. Shell commands can be executed during the building phase with the _shell_ directive, however, when using _ansible_ directives, _shell_ directives cannot be used and vice versa. Notice the commented _shell_ directive as an example.

Blocks inside the _ansible_ and _shell_ directives are known as Werf assembly instructions. Werf provides four user stages for executing these instructions: _beforeInstall_, _install_, _beforeSetup_, and _setup_. They are executed respectively, one after another. If you want to know more about user stages and the wide range of possibilities they offer, check [this documentation](https://werf.io/documentation/configuration/stapel_image/assembly_instructions.html#what-are-user-stages).

The _git_ directive is used to add files from the local Git repository into the image, but it can be used to add files from remote repositories too, allowing you to specify the branch, tag or commit to use, as well as path-based filters. Besides, this directive can be used to trigger the rebuilding of images when specific changes are made inside a Git repository. To see a comprehensive description of the directive and its capabilities, check [this documentation](https://werf.io/documentation/configuration/stapel_image/git_directive.html).

Finally, the _docker_ directive is used following a Dockerfile-like syntax in order to build the image. You can check the details of this directive [here](https://werf.io/documentation/configuration/stapel_image/docker_directive.html).

To build your docker image, you just need to execute the following command within the root level of your repository, so that the CLI can find the **werf.yaml** file. Note the _\--stages-storage_ flag, which indicates to Werf where to store the built [stages](https://werf.io/documentation/reference/stages_and_images.html#stages), which are the intermediate layers that are used to generate the image. In this case, by using the local flag, Werf will store the stages using the local docker runtime.

```bash
werf build --stages-storage :local
```

During the build process, the different stages are executed and their outputs are shown in the terminal.

![](images/Screen-Shot-2020-09-01-at-10.43.21-AM-1024x488.png)

Once the image is built successfully, you can push it to an images repository. For this example I used a [Dockerhub](https://hub.docker.com/) public repo for simplicity. If you don't have an account yet, just sign up and create a public repository. Once it's ready, generate an access token so that Werf can push the built images to the repository. To push the image, issue the following command.

```bash
werf publish --stages-storage :local --images-repo <your-repo>  --tag-custom v0.0.1  --images-repo-docker-hub-token  <your-token>
```

![](images/Screen-Shot-2020-08-31-at-3.56.49-PM.png)

**TIP:** If you prefer using a local Docker repository, rather than using Dockerhub, you can just run the following command, and specify _\--images-repo localhost:5000/your-repo-name_ when issuing the _werf publish_ command.

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

**TIP 2:** You can build and publish your image by just using one command.

```bash
werf build-and-publish  --stages-storage :local --images-repo <your-repo>  --tag-custom v0.0.1  --images-repo-docker-hub-token <your-token>
```

* * *

## Werf and secrets 🔐

Werf also comes in handy when working with sensitive values and secrets in Helm templates. Werf allows you to encrypt values files so that when they are pushed to a repository no one could extract the original values but Werf. To do so, Werf uses a secret key, which is specified using the _WERF\_SECRET\_KEY_ variable or the _.werf\_secret\_key_ file in the project root.

Generate a secret key and configure it as an environment variable for your shell session.

```bash
export WERF_SECRET_KEY=$(werf helm secret generate-secret-key)
```

Use Werf to generate the secret values file. This command will open a text editor so that you can create or modify the values.

```bash
werf helm secret values edit .helm/secret-values.yaml
```

Paste the following set of test values into the editor, then save your changes.

```yaml
server:
  hostname: werf-test
  user: kubesandclouds
```

If you try to read the content of the **.helm/secret-values.yaml** you will notice that the values have been encrypted, so now you're safe to store this secret values in your repository.

![](images/Screen-Shot-2020-08-30-at-1.15.54-AM-1024x122.png)

The secret values can be referenced as any other Helm value within the templates. For this example, they are injected in the container as environment variables.

```yaml
...
 env:
 - name: HOSTNAME
   value: {{ .Values.server.hostname }}
 - name: USER
   value: {{ .Values.server.user }}  
```

* * *

## Deploying to K8s ☸️

As commented before, Werf can use Helm templates to deploy applications to Kubernetes clusters. Werf searches for templates within **.helm/templates** and renders them as Helm would normally do. Besides, Werf extends Helm functionalities by including [three-way merge patches](https://medium.com/flant-com/3-way-merge-patches-helm-werf-beb7eccecdfe) as well as [resource tracking annotations](https://werf.io/documentation/reference/deploy_process/deploy_into_kubernetes.html#configuring-resource-tracking).

The code below shows the contents of [.helm/templates/deployment.yaml](https://github.com/mifonpe/werf-kubesandclouds/blob/master/.helm/templates/deployment.yaml), which uses the image previously build to deploy a three-replica web server in Kubernetes.

Notice how the go template functions _werf\_container\_image_ and _werf\_container\_env_ are used in the deployment manifest. These functions are evaluated when rendering the chart and generate the keys _image_ and _imagePullPolicy_ based on the tagging strategy, and the environment variable _DOCKER\_IMAGE\_ID_, respectively.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}-server
  labels:
    service: {{ .Chart.Name }}-server
spec:
  replicas: 3
  selector:
    matchLabels:
      service: {{ .Chart.Name }}-server
  template:
    metadata:
      labels:
        service: {{ .Chart.Name }}-server
    spec:
      containers:
      - name: web-server
{{ tuple "werf-web" . | werf_container_image | indent 8 }}
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        env:
        - name: HOSTNAME
          value: {{ .Values.server.hostname }}
        - name: USER
          value: {{ .Values.server.user }}   
{{ tuple "werf-web" . | werf_container_env | indent 8 }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-server
spec:
  clusterIP: None
  selector:
    service: {{ .Chart.Name }}-server
  ports:
  - name: http
    port: 8080
    targetPort: 80
    protocol: TCP
```

Like Helm, Werf uses the **.helm/requirements.yaml** file to define chart dependencies, like the nginx ingress controller in this case, to expose the web server. To build the dependencies, issue the following command.

```bash
werf helm dependency build
```

Once the dependencies have been fetched, deploy the application to your cluster. Werf uses the system default kubeconfig through the environment variable _KUBECONFIG_, but you can specify other kubeconfig by specifying its path with the _\--kube-config_ flag. The following command triggers the deployment process.

```bash
werf deploy  --images-repo <your-repo>  --stages-storage :local --tag-custom v0.0.1 --images-repo-docker-hub-token  <your-token>
```

![](images/Screen-Shot-2020-08-31-at-3.59.07-PM.png)

While Werf is performing the application deployment by means of Helm, the status progress is shown for all the components that are being deployed.

![](images/Screen-Shot-2020-08-31-at-3.59.32-PM-1024x240.png)

Once the deployment has been completed, a summary is displayed.

![](images/Screen-Shot-2020-08-31-at-4.00.17-PM.png)

If you deployed your application in your local cluster using Minikube or Docker desktop, execute the following command so that you can access your application through the ingress.

```bash
sudo -- sh -c 'echo "127.0.0.1 werfapp.local" >> /etc/hosts
```

If everything worked out, you should be able to see the example webpage displaying a pretty cool logo 😏 .

![](images/Screen-Shot-2020-09-01-at-1.33.10-PM-1024x389.png)

By the way, remember those super secret values we injected as environment variables? If you check the environment variables of the container within one of the pods, you can see how the values worked as expected.

![](images/Screen-Shot-2020-08-30-at-11.40.14-PM-1024x139.png)

**TIP 3:** The converge command combines _werf build_, _werf publish_ and _werf deploy_ commands. Notice that the _\--custom-tag_ flag is not used in this command, as the image tag is generated by Werf.

```bash
werf converge  --images-repo <your-repo>  --stages-storage :local  --images-repo-docker-hub-token <your-token>
```

![](images/Screen-Shot-2020-08-31-at-4.10.43-PM.png)

* * *

## Cleaning up 🧹

Once you're done playing around with the example application, it's time to remove the release and their associated resources. By executing the following command, the Helm release is removed from the cluster.

```bash
werf dismiss
```

In order to remove both the images pushed to the docker repository as well as the intermediate stages generated, issue the following command. Notice that for dockerhub repositories, both username and password need to be specified to delete images.

```bash
 werf  purge  --stages-storage :local --images-repo <your-repo> --repo-docker-hub-username <your-username> --repo-docker-hub-password <your-password>
```

**TIP 4:** The cleanup command can be used to delete unused images and stages. It is a good practice to run this command daily to keep just those images which are in use.

```bash
werf cleanup --stages-storage :local --images-repo <your-repo> --repo-docker-hub-username <your-username> --repo-docker-hub-password <your-password>
```

* * *

## CI/CD integrations 🐙🐱

As you have seen, Werf is a pretty powerful tool, and it makes sense to integrate it in a CI/CD environment to improve the overall automation and application lifecycle management. Integrating Werf is pretty simple if done 'manually', since it's shipped as a CLI, and its behavior can be configured by means of [environment variables](https://werf.io/documentation/reference/plugging_into_cicd/overview.html#pass-cli-parameters-as-environment-variables). Furthermore, Werf provides a command, [ci-env](https://werf.io/documentation/reference/plugging_into_cicd/overview.html), which detects the CI/CD system configuration and store it in the environment variables, making it even easier.

Besides, native integrations have been developed both for [GitLab CI](https://werf.io/documentation/guides/gitlab_ci_cd_integration.html) and [GitHub Actions](https://werf.io/documentation/guides/github_ci_cd_integration.html). The code below shows an example using actions provided by Werf to deploy into a Kubernetes cluster when new changes are pushed to the _deploy_ branch using GitHub actions. The action _werf/actions/converge@master_ executes the _werf converge_ command, building the images and deploying the application to the cluster pointed by the kubeconfig stored as a secret in _KUBE\_CONFIG\_BASE64\_DATA_.

```yaml
name: Staging Deployment
on:
  push:
    branches: [deploy]
jobs:
  converge:
    name: Converge
    runs-on: ubuntu-latest
    steps:

      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Install werf CLI  
        uses: werf/actions/install@master

      - name: Run werf commands
        run: |
          werf helm repo init
          werf helm dependency build

      - name: Converge
        uses: werf/actions/converge@master
        with:
          env: staging
          kube-config-base64-data: ${{ secrets.KUBE_CONFIG_BASE64_DATA }}
          
      - name: Dismiss if failed
        if: ${{ failure() }}
        uses: werf/actions/dismiss@master
        with:
          env: staging
          kube-config-base64-data: ${{ secrets.KUBE_CONFIG_BASE64_DATA }}
```

If you want to try this integration, you can fork the project and switch to the deploy branch which is already configured to be used with GitHub actions. In this case, if no Docker registry is specified, Werf will use GitHub packages as an image repository to store both the intermediate stages and the images.

![](images/Screen-Shot-2020-09-01-at-11.56.45-PM-1024x193.png)

In order to allow your Kubernetes cluster to pull images from this registry, you will need to create a Docker credentials secret specifying your username and a valid GitHub [Personal Access Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token), and then add the service account to use (the default in this case).

```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-token> 
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

If everything goes well, by the end of the job, Werf would have deployed the entire application in the cluster.

![](images/Screen-Shot-2020-09-02-at-12.15.28-AM-1024x738.png)

![](images/Screen-Shot-2020-09-02-at-12.18.55-AM-1024x194.png)

**TIP 5:** To obtain your kubeconfig in base64 you can use the following command (assuming your local kubeconfig points to the remote cluster to use).

```bash
kubectl config view --raw | base64
```

Thanks to these previously developed actions, integrating Werf in your existing CI/CD environment can be easier than you thought!

## Keep learning 👩‍💻👨‍💻

If you liked Werf, take some time to go through its [documentation](https://werf.io/documentation/), as you will better understand all the commands and options offered by the CLI, as well as all the possible integrations with CI/CD environments.

Besides, this documentation introduces advanced used cases that are really interesting as they present complex scenarios that resemble productive application environments and issues. Some of the most interesting use cases are the following ones:

- [Multi-images build](https://werf.io/documentation/guides/advanced_build/multi_images.html)
- [Image optimization by using mounts directive](https://werf.io/documentation/guides/advanced_build/mounts.html)
- [Image optimization using Werf artifacts](https://werf.io/documentation/guides/advanced_build/artifacts.html)

* * *

* * *

## Other Articles
