### Efficient and Secured way to access AWS Services from GKE
many of us have encountered multi-cloud projects where accessing resources across different cloud platforms becomes necessary. As I recently worked on a project that required secure and efficient access to AWS services from GKE pods, I want to share how I accomplished this integration.

Instead of following the naive and insecure approach of attaching static AWS credentials to the pods, which could compromise security, I explored alternative solutions.  
I'll walk you through a step by step guide of how I deployed external-dns on GKE and configured it to access AWS Route53.

Let's start with the AWS side.  
Using Terraform, I created an OIDC provider for GKE with the following configuration:
```bash
provider "aws" {
  region = var.aws_region
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
}

variable "gcp_region" {
  description = "GCP region"
  type        = string
}

variable "gcp_project_id" {
  description = "GCP project ID"
  type        = string
}
  
variable "gke_cluster_name" {
  description = "GKE cluster name"
  type        = string
}

data "tls_certificate" "gke_oidc" {
  url   = "https://container.googleapis.com/v1/projects/${var.gcp_project_id}/locations/${var.gcp_region}/clusters/${var.gke_cluster_name}"
}

resource "aws_iam_openid_connect_provider" "gke_oidc" {
  url   = "https://container.googleapis.com/v1/projects/${var.gcp_project_id}/locations/${var.gcp_region}/clusters/${var.gke_cluster_name}"

  client_id_list = [
    "sts.amazonaws.com"
  ]

  thumbprint_list = [
    data.tls_certificate.gke_oidc.certificates.0.sha1_fingerprint
  ]
}
```

This configuration establishes an OIDC provider on AWS that corresponds to the GKE cluster's API resource URL. By configuring the URL in this way, we essentially "trick" the pods into reaching out to this URL when they attempt to assume a role. However, instead of receiving the response directly from the GKE cluster, they will receive it from AWS.

Additionally, take note of the client ID (Identifier) of the AWS OIDC provider, which is set to "sts.amazonaws.com".  
When a pod requests a token from AWS, the request will automatically be directed to our AWS OIDC provider based on this client ID.

This setup ensures that pods in the GKE cluster can seamlessly assume roles and receive the necessary tokens from AWS using the OIDC provider.

Next, I created an AWS assumable role using Terraform modules:
```bash
module "iam_assumable_role_with_oidc_external_dns" {
  source                        = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version                       = "~> v5.9"
  create_role                   = true
  role_name                     = "external-dns-${var.gke_cluster_name}"
  role_description              = "External DNS role for GKE cluster ${var.gke_cluster_name}"
  provider_url                  = replace(aws_iam_openid_connect_provider.gke_oidc.url, "https://", "")
  role_policy_arns              = [aws_iam_policy.external_dns.arn]
  oidc_fully_qualified_subjects = ["system:serviceaccount:external-dns:external-dns"]
  tags                          = var.tags
}

data "aws_iam_policy_document" "external_dns" {
  statement {
    sid    = "ExternalDNSAllowChange"
    effect = "Allow"

    actions = [
      "route53:ChangeResourceRecordSets"
    ]

    resources = [
      "arn:aws:route53:::hostedzone/*"
    ]
  }

  statement {
    sid    = "ExternalDNSAllowList"
    effect = "Allow"

    actions = [
      "route53:ListHostedZones",
      "route53:ListResourceRecordSets"
    ]

    resources = ["*"]
  }
}

resource "aws_iam_policy" "external_dns" {
  name        = "gke-external-dns-${var.gke_cluster_name}"
  description = "External DNS policy for GKE cluster ${var.gke_cluster_name}"
  policy      = data.aws_iam_policy_document.external_dns.json
}
```

Note the **`oidc_fully_qualified_subjects`** section, which configures the OIDC provider to accept requests only from the "external-dns" service account in the "external-dns" namespace.

Moving on to the GCP side, we need to set specific environment variables for pods to assume the AWS role. In the case of Amazon EKS, AWS has developed a pod identity webhook that automatically sets these variables based on the service account's annotation.

Annotation:
```bash
metadata:
  annotations:
	eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_NUMBER>:role/<AWS_ROLE_NAME
```

On the GKE cluster, first, we need to deploy the EKS pod identity webhook.  
That can be achieved by applying resources from the EKS pod identity webhook [GitHub Repository](https://github.com/aws/amazon-eks-pod-identity-webhook/tree/master/deploy).

Now, let's deploy External-dns using Helm. 
Create a **`Values-dev.yaml`** file with the following content:
```bash
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_NUMBER>:role/<AWS_ROLE_NAME>
```

Finally, deploy everything with the Helm command:
```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm upgrade --install external-dns external-dns/external-dns -f values.yaml -n external-dns --create-namespace
```

With this setup, your GKE pods should be able to access AWS Services in a secured and efficient way.  
I hope you find this information valuable.

See you next time,

Yarden Weisman  
Cloud Native Engineer @ TeraSky
  
  
  
References:  
[EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook)  
[External DNS](https://github.com/kubernetes-sigs/external-dns)  
[Cross Account AWS IAM Assumable roles](https://docs.aws.amazon.com/eks/latest/userguide/cross-account-access.html)  