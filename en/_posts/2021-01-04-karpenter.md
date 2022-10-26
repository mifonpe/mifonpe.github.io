---
layout: post
title: Karpenter vs Cluster Autoscaler ☸️
subtitle: 
tags: [kubernetes, cloud, autoscaler]
comments: true
lang: en
lang-ref: karpenter
---

<p style='text-align: justify;'>Getting the size of a Kubernetes cluster right is not an easy task, if the number of nodes provisioned is too high, resources might be underutilized and if it&#8217;s too low, new workloads won&#8217;t be able to be scheduled in the cluster.</p>

<p style='text-align: justify;'>Setting the number of nodes manually is a simple approach, but it requires manual intervention every time the cluster needs to grow or to shrink, and it will make nearly impossible to adapt the cluster size to cover rapid traffic and load fluctuations. In order to solve this problem, the <a href="https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler" class="rank-math-link" target="_blank" rel="noopener">Cluster Autoscaler</a> was introduced starting on Kubernetes v1.4.</p>

<p style='text-align: justify;'>Around 4 years after the introduction of the Cluster Autoscaler, AWS started working on a new-generation Cluster Autoscaler: <a href="https://karpenter.sh/" class="rank-math-link" target="_blank" rel="noopener">Karpenter</a>. This article covers the fundamental differences between Cluster Autoscaler and Karpenter, followed by a demo that scales up an identical production-like cluster for a sample workload with 100 replicas.</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/karpenter_intro.jpeg" alt="Karpenter vs Cluster Autoscaler" sizes="(max-width: 1024px) 100vw, 1024px" /><figcaption>Karpenter vs Cluster Autoscaler</figcaption></figure>
</p>


<p style='text-align: justify;'>This is also the <strong>first collaborative article</strong> in <a href="https://kubesandclouds.com/" class="rank-math-link">Kubes&amp;Clouds</a>, where the main contributor is my colleague at <a href="https://www.sennder.com/" class="rank-math-link" target="_blank" rel="noopener">sennder</a> and friend <a href="https://www.linkedin.com/in/bhalothia/" class="rank-math-link" target="_blank" rel="noopener">Virendra Bhalothia</a>. He is a seasoned professional, who shares my passion for technology and knowledge sharing. You can check his own blog <a href="https://blog.bhalothia.io/" class="rank-math-link" target="_blank" rel="noopener">here</a>!</p>


<div class="wp-block-image" align="center"><figure class="aligncenter size-large is-resized"><img src= alt="/assets/img/vi.jpeg" width="512" height="383" sizes="(max-width: 512px) 100vw, 512px" /><figcaption>Virendra and I saving the world</figcaption></figure></div>

<h2>AWS Announcement at re:Invent 2021 🟠☁️</h2>

<p style='text-align: justify;'><a href="https://karpenter.sh/" class="rank-math-link" target="_blank" rel="noopener">Karpenter</a> is an open-source, flexible, high-performance Kubernetes cluster autoscaler built by AWS, that was <a href="https://aws.amazon.com/blogs/aws/introducing-karpenter-an-open-source-high-performance-kubernetes-cluster-autoscaler/" target="_blank" rel="noopener">released</a> (GA version) at <em>re:Invent 2021</em>.</p>

<p style='text-align: justify;'>This means it&#8217;s officially ready for production workloads as per AWS. However, it&#8217;s been discussed for almost a year. Some of the <a href="https://github.com/aws/karpenter#talks" class="rank-math-link" target="_blank" rel="noopener">conference talks</a> can be found on the Karpenter&#8217;s <a href="https://github.com/aws/karpenter" class="rank-math-link" target="_blank" rel="noopener">Github repo</a>.</p>

<p style='text-align: justify;'>The official blog announcement promises support for K8s clusters running in any environment. However, it currently only supports AWS as cloud provider, but do keep an eye on the <a href="https://github.com/aws/karpenter/projects/3" target="_blank" rel="noopener">Karpenter project roadmap</a> if you use other underlying cloud providers or on-premises datacenters.</p>

<h2><strong>Kubernetes Autoscaling Capabilities</strong> ☸️ 🎛️</h2>

<p style='text-align: justify;'>Alright! So, before we get started with comparing Cluster Autoscaler with Karpenter let&#8217;s quickly go over the autoscaling capabilities Kubernetes offers. <span class="has-inline-color has-very-dark-gray-color">Kubernetes enables autoscaling at the node level as well as at the pod level. These two are different but fundamentally connected layers of Kubernetes architecture.</span> </p>

<p style='text-align: justify;'><strong>Pod-based autoscaling</strong></p>

<ul><li> <a href="https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/" class="rank-math-link" target="_blank" rel="noopener">Horizontal Pod Autoscaler</a> &#8211; adds or removes more pods to the deployment as needed</li><li> <a href="https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler" class="rank-math-link" target="_blank" rel="noopener">Vertical Pod Autoscaler</a> &#8211; resizes pod&#8217;s CPU and memory requests and limits to match the load</li></ul>

<p style='text-align: justify;'><strong>Node-based autoscaling</strong>: adding or removing nodes as needed</p>

<ul><li><a href="https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler" class="rank-math-link" target="_blank" rel="noopener">Cluster Autoscaler</a></li><li><a href="https://karpenter.sh/docs/getting-started/" class="rank-math-link" target="_blank" rel="noopener">Karpenter</a></li></ul>

<p style='text-align: justify;'><strong>NOTE</strong>: This post will only explore the node-based Kubernetes Autoscalers.</p>

<h2><strong>Cluster Autoscaler vs Karpenter</strong> ⚒️🤖</h2>



<p style='text-align: justify;'><a href="https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#introduction" class="rank-math-link" target="_blank" rel="noopener">Cluster Autoscaler</a> is a Kubernetes tool that increases or decreases the size of a Kubernetes cluster (by adding or removing nodes), based on the presence of pending pods and node utilization metrics.</p>



<p style='text-align: justify;'>It automatically adjusts the size of the Kubernetes cluster when one of the following conditions is true:</p>

<ul><li>There are pods that failed to run in the cluster due to insufficient resources</li><li>There are nodes in the cluster that have been underutilized for an extended period of time and their pods can be placed on other existing nodes</li></ul>

<p style='text-align: justify;'><a href="https://github.com/aws/karpenter#readme" class="rank-math-link" target="_blank" rel="noopener">Karpenter</a> automatically provisions new nodes in response to unschedulable pods. Karpenter does this by observing events within the Kubernetes cluster, and then sending commands to the underlying cloud provider. </p>

<p style='text-align: justify;'>Karpenter works by:</p>

<ul><li>Watching for pods that the Kubernetes scheduler has marked as unschedulable</li><li>Evaluating scheduling constraints (resource requests, nodeselectors, affinities, tolerations, and topology spread constraints) requested by the pods</li><li>Provisioning nodes that meet the requirements of the pods</li><li>Scheduling the pods to run on the new nodes and removing the nodes when the nodes are no longer needed<br></li></ul>

<p style='text-align: justify;'>Karpenter has two control loops that maximize the availability and efficiency of your cluster.</p>

<ul><li><strong>Allocator</strong> &#8211; fast-acting controller ensuring that pods are scheduled as quickly as possible</li><li><strong>Reallocator</strong> &#8211; slow-acting controller replaces nodes as pods capacity shifts over time</li></ul>

<h3><strong>Architecture</strong> 🏗️</h3>

<p style='text-align: justify;'>Cluster Autoscaler watches for pods that fail to schedule and for nodes that are underutilized. It then simulates the addition or removal of nodes before applying the change to your cluster. </p>

<p style='text-align: justify;'>The AWS Cloud Provider implementation within Cluster Autoscaler<strong> </strong>controls the <em>DesiredReplicas</em> field of your EC2 Auto Scaling Groups. The Kubernetes Cluster Autoscaler automatically adjusts the number of nodes in your cluster when pods fail or are rescheduled onto other nodes. The Cluster Autoscaler is typically installed as a Deployment in your cluster. It uses a leader election algorithm to ensure high availability, but scaling is done by only one replica at a time.</p>


<p align="center">
<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2021/12/cluster_autoscaler.png" alt="" class="wp-image-2497" srcset="https://kubesandclouds.com/wp-content/uploads/2021/12/cluster_autoscaler.png 741w, https://kubesandclouds.com/wp-content/uploads/2021/12/cluster_autoscaler-300x157.png 300w" sizes="(max-width: 741px) 100vw, 741px" /><figcaption>Cluster Autoscaler</figcaption></figure></div>
</p>


<p style='text-align: justify;'>On the other hand, Karpenter works in tandem with the Kubernetes scheduler by observing incoming pods over the lifetime of the cluster. It launches or terminates nodes to maximize application availability and cluster utilization. When there is enough capacity in the cluster, the Kubernetes scheduler will place incoming pods as usual. </p>

<p style='text-align: justify;'>When pods are launched that cannot be scheduled using the existing capacity of the cluster, <strong>Karpenter bypasses the Kubernetes scheduler and works directly with your provider’s compute service</strong>, (for example, AWS EC2), to launch the minimal compute resources needed to fit those pods and binds the pods to the nodes provisioned. As pods are removed or rescheduled to other nodes, Karpenter looks for opportunities to terminate under-utilized nodes.</p>



<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2021/12/karpenter-1024x457.png" alt="" class="wp-image-2500" srcset="https://kubesandclouds.com/wp-content/uploads/2021/12/karpenter-1024x457.png 1024w, https://kubesandclouds.com/wp-content/uploads/2021/12/karpenter-300x134.png 300w, https://kubesandclouds.com/wp-content/uploads/2021/12/karpenter-768x343.png 768w, https://kubesandclouds.com/wp-content/uploads/2021/12/karpenter.png 1052w" sizes="(max-width: 1024px) 100vw, 1024px" /><figcaption>Karpenter</figcaption></figure></div>

<p style='text-align: justify;'>Detailed Kubernetes Autoscaling guidelines from AWS can be found <a class="rank-math-link" href="https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html" target="_blank" rel="noopener">here</a>.</p>

<h3><strong>Karpenter improvements</strong></h3>

<ul><li><strong>Designed to handle the full flexibility of the cloud</strong>: Karpenter has the ability to efficiently address the full range of instance types available through AWS. Cluster autoscaler was not originally built with the flexibility to handle hundreds of instance types, zones, and purchase options.</li><li><strong>Group-less node provisioning</strong>: Karpenter manages each instance directly, without using additional orchestration mechanisms like node groups. This enables it to retry in milliseconds instead of minutes when capacity is unavailable. It also allows Karpenter to leverage diverse instance types, availability zones, and purchase options without the creation of hundreds of node groups.</li><li><strong>Scheduling enforcement:</strong> Cluster autoscaler<strong> </strong>doesn’t bind pods to the nodes it creates. Instead, it relies on the kube-scheduler to make the same scheduling decision after the node has come online. A node that Karpenter launches has its pods bound immediately. The kubelet doesn’t have to wait for the scheduler or for the node to become ready. It can start preparing the container runtime immediately, including pre-pulling the image. This can shave seconds off of node startup latency.</li></ul>

<h2><strong>Implementing it</strong> ➡️ ⚒️</h2>

<p style='text-align: justify;'>In this section, we will create the required IAM &amp; K8s resources for Karpenter using Terraform and Helm. Once we are all set up then we will give it a go and check how quickly Karpenter scales up your cluster as compared to the Cluster Autoscaler.</p>

<p style='text-align: justify;'>Make sure the following tools are installed in your local environment before proceeding:</p>

<ul><li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html" class="rank-math-link" target="_blank" rel="noopener">AWS CLI</a></li><li><a href="https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/" class="rank-math-link" target="_blank" rel="noopener">kubectl </a></li><li><a href="https://learn.hashicorp.com/tutorials/terraform/install-cli" class="rank-math-link" target="_blank" rel="noopener">terraform</a></li><li><a href="https://helm.sh/docs/intro/install/" class="rank-math-link" target="_blank" rel="noopener">helm</a></li></ul>

<p style='text-align: justify;'><span class="has-inline-color has-very-dark-gray-color">We are assuming that the <strong>underlying VPC and network resources are already created along with the EKS cluster as well as the Cluster Autoscaler</strong>, and we will add Karpenter resources on top of that.</span> In case you haven&#8217;t created them, you can check <a href="https://github.com/mifonpe/argocd-gitops-demo/tree/main/iac" class="rank-math-link" target="_blank" rel="noopener">this Terraform code</a>, from a <a href="https://www.youtube.com/watch?v=maepNvKkO7o" class="rank-math-link" target="_blank" rel="noopener">webinar</a> in which I participated. For the Cluster Autoscaler, you can find its installation guide for EKS <a href="https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#cluster-autoscaler" class="rank-math-link" target="_blank" rel="noopener">here</a>.</p>

<p style='text-align: justify;'><strong>NOTE</strong>: Please note that Karpenter expects the subnets to be tagged with <em><strong>&#8220;kubernetes.io/cluster/&lt;cluster-name&gt;&#8221; = &#8220;owned</strong></em>&#8220;.</p>

<p style='text-align: justify;'>If you are using Terraform then it&#8217;s as simple as adding this one line in your <a href="https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest" class="rank-math-link" target="_blank" rel="noopener">VPC module</a>.</p>


```json
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  ...
  private_subnet_tags = {
    "kubernetes.io/cluster/&lt;your-cluster-name-here>" = "owned"
  }
  ...
}
```

<h3><strong>Configure the KarpenterNode IAM Role</strong></h3>

<p style='text-align: justify;'>The first step will be to create a <em>karpenter.tf</em>  file in your terraform directory to add the snippet below.<br><br>The EKS module creates an IAM role for worker nodes. We’ll use that for Karpenter (so we don’t have to reconfigure the <a href="https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html" class="rank-math-link" target="_blank" rel="noopener">aws-auth ConfigMap</a>), but we need to add one more policy and create an instance profile.</p>


```json
data "aws_iam_policy" "ssm_managed_instance" {
  arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "karpenter_ssm_policy" {
  role       = module.eks.worker_iam_role_name
  policy_arn = data.aws_iam_policy.ssm_managed_instance.arn
}

resource "aws_iam_instance_profile" "karpenter" {
  name = "KarpenterNodeInstanceProfile-&lt;your-cluster-name>"
  role = module.eks.worker_iam_role_name
}
```



<p style='text-align: justify;'><span class="has-inline-color has-very-dark-gray-color">Now, Karpenter can use this instance profile to launch new EC2 instances and those instances will be able to connect to your cluster.</span></p>

<h3>Configure the KarpenterController IAM Role</h3>

<p style='text-align: justify;'>Karpenter requires permissions for launching instances, which means it needs an IAM role that is granted that access. The config below will create an AWS IAM Role, attach a policy, and authorize the Service Account to assume the role using IRSA using an <a href="https://github.com/terraform-aws-modules/terraform-aws-iam/tree/v4.8.0/modules/iam-assumable-role-with-oidc" class="rank-math-link" target="_blank" rel="noopener">AWS terraform module</a>. We will create the ServiceAccount and connect it to this role during the Helm chart installation.</p>


```json
module "iam_assumable_role_karpenter" {
  source                        = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version                       = "4.7.0"
  create_role                   = true
  role_name                     = "karpenter-controller-&lt;your-cluster-name>"
  provider_url                  = module.eks.cluster_oidc_issuer_url
  oidc_fully_qualified_subjects = ["system:serviceaccount:karpenter:karpenter"]
}

resource "aws_iam_role_policy" "karpenter_contoller" {
  name = "karpenter-policy-&lt;your-cluster-name>"
  role = module.iam_assumable_role_karpenter.iam_role_name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:CreateLaunchTemplate",
          "ec2:CreateFleet",
          "ec2:RunInstances",
          "ec2:CreateTags",
          "iam:PassRole",
          "ec2:TerminateInstances",
          "ec2:DescribeLaunchTemplates",
          "ec2:DescribeInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeInstanceTypeOfferings",
          "ec2:DescribeAvailabilityZones",
          "ssm:GetParameter"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]
  })
}
```

<h3><strong>Install Karpenter Helm Chart</strong></h3>

<p style='text-align: justify;'>Now you can use Helm to deploy Karpenter to the cluster. While installing the chart, we will override some of the default values with the cluster specific values, so that Karpenter can work properly in our cluster. In this case, an in-line call to the AWS CLI is used to retrieve the cluster endpoint.</p>

```bash
>~ helm repo add karpenter https://charts.karpenter.sh
~ helm repo update
~ helm upgrade --install karpenter karpenter/karpenter --namespace karpenter \
--create-namespace --set serviceAccount.create=true --version 0.5.3 \
--set serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn=${IAM_ROLE_ARN} \
--set controller.clusterName=${CLUSTER_NAME} \
--set controller.clusterEndpoint=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${REGION} --profile ${AWS_PROFILE} --query "cluster.endpoint" --output json) --wait
```
<p style='text-align: justify;'>This should create the following resources in the <em>karpenter</em> namespace.</p>

```bash
~ kubectl get all
NAME                                        READY   STATUS    RESTARTS   AGE
pod/karpenter-controller-64754574df-gqn86   1/1     Running   0          29s
pod/karpenter-webhook-7b88b965bc-jcvhg      1/1     Running   0          29s

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/karpenter-metrics   ClusterIP   10.x.x.x   &lt;none>        8080/TCP   29s
service/karpenter-webhook   ClusterIP   10.x.x.x    &lt;none>        443/TCP    29s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/karpenter-controller   1/1     1            1           30s
deployment.apps/karpenter-webhook      1/1     1            1           30s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/karpenter-controller-64754574df   1         1         1       30s
replicaset.apps/karpenter-webhook-7b88b965bc      1         1         1       30s
```

<p style='text-align: justify;'>Also, it should create a Service Account.</p>

```bash
~ kubectl describe sa karpenter -n karpenter
Name:                karpenter
Namespace:           karpenter
Labels:              app.kubernetes.io/managed-by=Helm
Annotations:         eks.amazonaws.com/role-arn: &lt;Obfuscated_IAM_Role_ARN>
                     meta.helm.sh/release-name: karpenter
                     meta.helm.sh/release-namespace: karpenter
Image pull secrets:  image-pull-secret
Mountable secrets:   karpenter-token-dwwrs
Tokens:              karpenter-token-dwwrs
Events:              &lt;none>
```

<h3><strong>Configure a Karpenter Provisioner</strong></h3>

<p style='text-align: justify;'>A single <a href="https://karpenter.sh/docs/provisioner/" class="rank-math-link" target="_blank" rel="noopener">Karpenter provisioner</a> is capable of handling many different pod types. Karpenter makes scheduling and provisioning decisions based on pod attributes such as labels and affinity. In other words, Karpenter eliminates the need to manage many different node groups.</p>

<p style='text-align: justify;'>Create a default provisioner using the command below. This provisioner configures instances to connect to your cluster&#8217;s endpoint and discovers resources like subnets and security groups using the cluster&#8217;s name.</p>

```bash
cat &lt;&lt;EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  provider:
    instanceProfile: KarpenterNodeInstanceProfile-${CLUSTER_NAME}
  ttlSecondsAfterEmpty: 30
EOF
```

<p style='text-align: justify;'>The&nbsp;<em>ttlSecondsAfterEmpty</em> value configures Karpenter to terminate empty nodes. This behavior can be disabled by leaving the value undefined.</p>

<p style='text-align: justify;'>Review the <a href="https://karpenter.sh/docs/provisioner/" class="rank-math-link" target="_blank" rel="noopener">provisioner CRD </a>for more information. For example, <em>ttlSecondsUntilExpired</em> configures Karpenter to terminate nodes when a maximum age is reached.</p>

<p style='text-align: justify;'>Alright, so now we have a Karpenter provisioner that supports Spot <a href="https://karpenter.sh/docs/provisioner/#capacity-type" class="rank-math-link" target="_blank" rel="noopener">capacity type</a>. <strong>In a real-world scenario, you might have to manage a variety of Karpenter provisioners that can support your workloads</strong>. </p>


<h2><strong>Testing it</strong> 🧪🔥</h2>

<p style='text-align: justify;'><span class="has-inline-color has-very-dark-gray-color">Now, all what is left to do is to deploy a sample app and see how it scales via Cluster Autoscaler and Karpenter respectively.</span> To do so, you can execute the following command.</p>

```bash
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.2
```

<p style='text-align: justify;'>Also, let&#8217;s set some resource requests for this vanilla<em> inflate</em> deployment:</p>


```bash
kubectl set resources deployment inflate --requests=cpu=100m,memory=256Mi
```

<h3>Cluster Autoscaler</h3>


<p style='text-align: justify;'>It is important to scale down the Karpenter deployment to 0 replicas before increasing the number of  <em>inlflate</em> replicas, so that Cluster Autoscaler handles the addition of new nodes.</p>



```bash
kubectl scale deployment karpenter-controller -n karpenter --replicas=0
```

<p style='text-align: justify;'>Now, let&#8217;s scale the inflate deployment up to 100 replicas. <span class="has-inline-color has-vivid-red-color"><strong>Please note this may incur costs to your AWS cloud bill, so be careful. </strong></span></p>


```bash
kubectl scale deployment inflate --replicas 100
```

<p style='text-align: justify;'>Cluster Autoscaler works with node groups in order to scale out or in as per the dynamic workloads. In order to get efficient autoscaling with Cluster Autoscaler, there are a lot of considerations that need to be reviewed and applied accordingly to the requirements. You can find more details about this topic <a href="https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html" class="rank-math-link" target="_blank" rel="noopener">here</a>. This demo was run on a test cluster that had an ASG with identical instance specifications with the instance type <em>c5.large</em>.</p>


<figure class="wp-block-image size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2021/12/Screenshot-2021-12-28-at-14.28.33-1024x525.png" alt="" class="wp-image-2514" /><figcaption>Inflate workload with Cluster autoscaler</figcaption></figure>

<p style='text-align: justify;'>The <em>inflate</em> deployment kept on waiting for the Cluster Autoscaler to schedule all the pods for around 4 minutes. If you execute<em> kubectl get nodes </em>during the scaling process, you will see the new nodes kicking in.</p>


<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-12.40.24-1.png" alt="" class="wp-image-2641" sizes="(max-width: 629px) 100vw, 629px" /></figure></div>


<p style='text-align: justify;'>As commented before, Cluster Autoscaler operates on the autoscaling groups. You can see how the number of instances of the targeted autoscaling group is increased using the AWS console.</p>


<figure class="wp-block-image size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-12.42.42-1024x161.png" alt="" class="wp-image-2611" sizes="(max-width: 1024px) 100vw, 1024px" /></figure>

<p style='text-align: justify;'>It&#8217;s also possible to check the autoscaling events in the console. These events were triggered by the Cluster Autoscaler, calling the EC2 API.</p>


<figure class="wp-block-image size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-12.50.24-1024x283.png" alt="" class="wp-image-2612" sizes="(max-width: 1024px) 100vw, 1024px" /></figure>

<p style='text-align: justify;'>Finally, let&#8217;s scale down the inflate deployment back to one replica. </p>

```bash
kubectl scale deployment inflate --replicas 1</code></pre>
```

<p style='text-align: justify;'>In my case, it took the Cluster Autoscaler 6 minutes to trigger a downscaling event and start reducing the cluster size. You can check the logs of the Cluster Autoscaler to see how the autoscaling process happens.</p>


<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-12.54.22-1.png" alt="" class="wp-image-2639" sizes="(max-width: 644px) 100vw, 644px" /></figure></div>

<h3>Karpenter</h3>

<p style='text-align: justify;'>Let&#8217;s repeat the experiment now, but using Karpenter this time. To do so, <strong>scale down your Cluster Autoscaler and scale up the Karpenter Controller.</strong></p>


```bash
kubectl scale deployment cluster-autoscaler -n autoscaler --replicas=0
kubectl scale deployment karpenter-controller -n karpenter --replicas=1
```

<p style='text-align: justify;'>Now that Karpenter is taking care of the autoscaling of the cluster, scale the inflate deployment to 100 replicas again and monitor the logs of the Karpenter controller.</p>

```bash
kubectl scale deployment inflate --replicas 100
```

<p style='text-align: justify;'>The following snippet contains part of the karpenter-controller logs after the deployment was scaled up. It&#8217;s using the default provisioner that was defined in the previous version, and it has a wide ranges of instances to select from, as we did not specify any specific instance type in the provisioner.</p>

```bash
2021-12-28T14:23:58.816Z INFO controller.provisioning Batched 89 pods in 4.587455594s {"commit": "5047f3c", "provisioner": "default"}

2021-12-28T14:23:58.916Z INFO controller.provisioning Computed packing of 1 node(s) for 89 pod(s) with instance type option(s) [m5zn.3xlarge c3.4xlarge c4.4xlarge c5ad.4xlarge c5a.4xlarge c5.4xlarge c5d.4xlarge c5n.4xlarge m5ad.4xlarge m5n.4xlarge m5.4xlarge m5a.4xlarge m6i.4xlarge m5d.4xlarge m5dn.4xlarge m4.4xlarge r3.4xlarge r4.4xlarge r5b.4xlarge r5d.4xlarge] {"commit": "5047f3c", "provisioner": "default"}

2021-12-28T14:24:01.013Z INFO controller.provisioning Launched instance: i-xx, hostname: ip-10-x-x-x.eu-x-1.compute.internal, type: c5.4xlarge, zone: eu-x-c, capacityType: spot {"commit": "5047f3c", "provisioner": "default"}

2021-12-28T14:24:01.222Z INFO controller.provisioning Bound 89 pod(s) to node ip-10-x-x-x.eu-x-1.compute.internal {"commit": "5047f3c", "provisioner": "default"}

2021-12-28T14:24:01.222Z INFO controller.provisioning Waiting for unschedulable pods {"commit": "5047f3c", "provisioner": "default"}
```



<p style='text-align: justify;'>As you can see in the log above, Karpenter binds the pods to the newly provisoned node, but how is it done? If you check any of the pod&#8217;s definition for the <em>inflate</em> deployment, you will notice that the <a href="https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename" class="rank-math-link" target="_blank" rel="noopener">nodeName</a> field points to the new node.</p>

<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-13.24.03-1.png" alt="" class="wp-image-2638"  sizes="(max-width: 680px) 100vw, 680px" /></figure></div>

<p style='text-align: justify;'>The instance provisioned is not a part of an autoscaling group, and in this case it&#8217;s a spot c5.4xlarge instance. The instance type was selected by Karpenter internal algorithm, but you can customize the provisioner to use the instance types that better suit your needs with the <strong>n<em>ode.kubernetes.io/instance-type</em></strong><em> </em>directive<em> . </em>Check the<a href="https://karpenter.sh/docs/provisioner/"  target="_blank" rel="noopener"> provisioner API</a> to<em> </em>get more information about how to customize your provisioners.</p>

<div class="wp-block-image"><figure class="aligncenter size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2022/01/Screenshot-2022-01-04-at-13.08.34-1024x149.png" alt="" class="wp-image-2623" sizes="(max-width: 1024px) 100vw, 1024px" /></figure></div>

<p style='text-align: justify;'>So, basically, Karpenter detects there are some unschedulable pods in the cluster. It does the math and provisions the best-suited spot instance from the available options.  It took around 2 minutes for the <em>inflate</em> deployment with 100 replicas to be fully deployed.</p>


<figure class="wp-block-image size-large"><img src="https://kubesandclouds.com/wp-content/uploads/2021/12/Screenshot-2021-12-28-at-14.51.45-1024x493.png" alt="" class="wp-image-2515" sizes="(max-width: 1024px) 100vw, 1024px" /><figcaption>Inflate Workload with Karpenter</figcaption></figure>

<p style='text-align: justify;'>Finally, let&#8217;s wrap it up by scaling down the deployment to 0 replicas.</p>

```bash
kubectl scale deployment inflate --replicas 0
```

<p style='text-align: justify;'>If you check the logs, you will see how Karpenter deprovisions the instance right away.</p>

```bash
2021-12-28T14:31:12.364Z INFO controller.node Added TTL to empty node {"commit": "5047f3c", "node": "ip-10-x-x-x.eu-x-1.compute.internal"}

2021-12-28T14:31:42.391Z INFO controller.node Triggering termination after 30s for empty node {"commit": "5047f3c", "node": "ip-10-x-x-x.eu-x-1.compute.internal"}

2021-12-28T14:31:42.418Z INFO controller.termination Cordoned node {"commit": "5047f3c", "node": "ip-10-x-x-x.eu-x-1.compute.internal"}

2021-12-28T14:31:42.620Z INFO controller.termination Deleted node {"commit": "5047f3c", "node": "ip-10-x-x-x.eu-x-1.compute.internal"}
```

<p style='text-align: justify;'>It took 30 seconds for Karpenter to terminate the node once there was no pod scheduled on it.</p>

<h2>Conclusion 📖🧑‍🏫</h2>

<p style='text-align: justify;'>The Karpenter project is exciting and breaks away from the old school Cluster Autoscaler way of doing things. It is very efficient and fast but not as much battle-tested as the &#8216;good old&#8217; Cluster Autoscaler.</p>


<p style='text-align: justify;'><span>Please note that the above experiment with Cluster Autoscaler could be greatly improved if we set up the right instance types and autoscaling policies. However, that&#8217;s the whole point of this comparison: <strong>with Karpenter you don&#8217;t have to ensure all of these configurations beforehand. You might end up having a lot of provisioners for your different workloads</strong> but that will be covered in upcoming posts for this series.</span></p>


<p style='text-align: justify;'>If you are running Kubernetes then I’m sure you will enjoy spinning it up and playing around with it. Good luck!</p>


<hr class="wp-block-separator"/>



<h2>Next Steps 👩‍💻👨‍💻</h2>



<p style='text-align: justify;'>Karpenter is an open source project with a <a href="https://github.com/aws/karpenter/projects" class="rank-math-link" target="_blank" rel="noopener">roadmap</a> open to all, so it&#8217;s definitely worth keeping an eye out on it, as well as on the <a href="https://github.com/aws/karpenter/issues" class="rank-math-link" target="_blank" rel="noopener">issues</a> that people might face.</p>



<p style='text-align: justify;'>As an extension to this experiment, we will be writing an article based on the experience of <strong>migrating a production grade cluster from Cluster Autoscaler to Karpenter</strong>, as well as <strong>in-depth performance comparison</strong> article, so expect more cool stuff on this series!</p>


