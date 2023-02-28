---
title: "Terraboard: Graphic Terraform state manager🌍🖥"
date: "2020-11-18"
categories: 
  - "aws"
  - "iac"
tags: 
  - "aws"
  - "iac"
  - "terraform"
lang: es
lang-ref: terraboard
---

Dealing with multiple terraform remote states can become a rather complex task. Besides, querying resources with terraform CLI isn't very visual 😅. In this post we will present [Terraboard](https://camptocamp.github.io/terraboard/), an open source tool developed by [Camptocamp](https://www.camptocamp.com/en) that solves these issues.

![](/assets/img/imported/Icogram-2020-11-20-19_37-1-1-1024x512.png)

Terraboard provides a web interface which also adds a diff tool to compare different resource versions. It currently supports terraform states in both AWS S3 and Terraform Cloud backends.

* * *

## Testing it locally 💻

Terraboard is distributed as a CLI written in Go, alongside an AngularJS web UI, and it requires a Postgres database in order to work. There are several ways to test it and deploy it, as you can check in [Terraboard's GitHub Repo](https://github.com/camptocamp/terraboard). For this example, we will use Terraboard's Docker image. Besides, [a simple script](https://github.com/mifonpe/terraboard) has been developed to make it easier to run Terraboard locally.

In order to execute this script, Docker must be installed in your local machine. You can get the appropriate version for your OS [here](https://docs.docker.com/get-docker/). Execute the following commands to download it and so that you can invoke the script from your command line.

```bash
curl -fsSL -o terraboard https://raw.githubusercontent.com/mifonpe/terraboard/main/terraboard
chmod 700 terraboard
mv terraboard /usr/local/bin
```

For this example, AWS S3 will be used as the backend, so you will need a pair of access key and secret access key. If you don’t have one yet, you can create a new one in the AWS IAM console. Check [this documentation](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys) as a guide.

Keep in mind that the IAM user associated to the keys should have the following set of permissions on the backend s3 bucket:

- s3:GetObject
- s3:ListBucket
- s3:ListBucketVersions
- s3:GetObjectVersion

If you want to limit the scope of this programmatic user so that it only has access to the state bucket, you can use the following IAM policy adding the name of the bucket which holds the terraform states.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket" ],
      "Resource": ["arn:aws:s3:::your-bucket-name"]
    },
    {
      "Effect": "Allow",
      "Action": [
         "s3:ListBucket",
         "s3:ListBucketVersions",
         "s3:GetObject",
         "s3:GetObjectVersion"
      ],
      "Resource": ["arn:aws:s3:::your-bucket-name/*"]
    }
  ]
}
```

Keep in mind that the terraform bucket needs to be versioned so that Terraboard can compare the different versions of the states.

Once the keys and permissions are set, export them as environment variables as well as the AWS region and bucket name.

```bash
export AWS_ACCESS_KEY_ID=<your-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export AWS_REGION=<your-region>
export TFSTATE_BUCKET=<your-bucket>
```

Start Terraboard and wait for the script to complete.

```
terraboard up
```

![](/assets/img/imported/Screen-Shot-2020-11-19-at-8.48.13-PM.png)

If everything went well and you hit localhost on port 8080 you will be able to access Terraboard's UI. In this case it's showing the different terraform state files present in the bucket. Feel free to play around with the interface, and give a look to the search utility, which allows filtering resources by different criteria such as Terraform version or resource type.

![](/assets/img/imported/Screen-Shot-2020-11-20-at-6.02.42-PM-1024x461.png)

If you examine a specific state, you can go through its resources and modules, getting detailed information and parameters.

![](/assets/img/imported/Screen-Shot-2020-11-20-at-2.23.22-AM-1-1024x538.png)

As commented before, Terraboard allows comparing different state versions, which can come in handy to detect the root cause of incidents within the infrastructure.

![](/assets/img/imported/Screen-Shot-2020-11-20-at-2.23.40-AM-1-1024x541.png)

As you can see in the image below, Terraboard provides a _git-like_ diff for the state files.

![](/assets/img/imported/Screen-Shot-2020-11-20-at-2.24.27-AM-1024x424.png)

Once you're done playing around with Terraboard, you can remove the containers running in your local machine and the associated resources by issuing the following command.

```
terraboard down
```

* * *

## Deploying it to Kubernetes ☸️

In order to make Terraboard accessible to your entire team or other collaborators, you can deploy it to a Kubernetes cluster. A [simple Helm chart](https://github.com/mifonpe/terraboard/tree/main/helm) has been developed for that purpose. To get it deployed, execute the following commands, providing your AWS access key and secret key as extra arguments.

```bash
git clone https://github.com/mifonpe/terraboard
cd terraboard/helm
helm install terraboard . --set secrets.aws_key=<your-key> \
--set secrets.aws_secret_key=<your-access-key> \
--set backend.region=<aws-region> \
--set backend.bucket=<your-bucket>
```

**NOTE:** this Helm chart assumes that a Nginx ingress controller is already in place. If you don't have an ingress controller in place, install it with the following command.

```bash
helm install ingress bitnami/nginx-ingress-controller
```

* * *
