# Reusing One AWS IAM Role Across Multiple vClusters

*A hands-on lab that answers a real support question: "Can multiple tenant clusters share the same AWS permission, instead of creating a new one every time?"*

---

## Table of Contents

1. [The Problem We're Solving](#1-the-problem-were-solving)
2. [Glossary — Every Term Explained Simply](#2-glossary--every-term-explained-simply)
3. [The Big Idea Before We Start](#3-the-big-idea-before-we-start)
4. [Prerequisites](#4-prerequisites)
5. [Step by Step](#5-step-by-step)

---

## 1. The Problem We're Solving

Imagine a company runs many small, isolated "vCluster" inside one big Kubernetes cluster. Each of these vClusters needs to regularly back up its data to Amazon S3 (cloud file storage).

To let a vCluster talk to S3, AWS requires it to prove its identity first, like showing an ID card before entering a building. The normal way to set this up is:

1. Create an AWS permission (called an IAM role).
2. Explicitly link that one vCluster to that one role.

The question this lab answers is: **if a company creates 50 of these vClusters, do they need to repeat step 2 fifty times, once per vCluster, or can they set things up once and have it apply automatically to every new vCluster, forever?**

I tested this for real, in a real AWS account, and got a clear, tested answer. This guide walks through exactly how.

---

## 2. Prerequisites

This lab assumes you already have the generic, standard pieces in place, since setting these up is well covered elsewhere and isn't specific to this trick:

- A working **EKS cluster**, with at least one node group up and running (see AWS's own EKS getting-started guide if you need this first)
- `aws`, `kubectl`, and `helm` installed locally, with `aws configure` already pointing at your account
- **An Enterprise license for vCluster Platform.** Scheduled snapshots are an Enterprise feature, not a free one.

---

## 3. Step by Step

### Step 1: Create the S3 Bucket and IAM Policy

Create an S3 bucket you want tenant clusters to back up into. From the S3 Console (**Create bucket**, give it a unique name, same region as your cluster, defaults are fine), or from the CLI:

```bash
aws s3 mb s3://your-bucket-name --region <your-region>
```

Then create an IAM policy granting access to that one bucket:

**IAM Console → Policies → Create policy → JSON tab**, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

Name it `snapshot-s3-access-policy`.

### Step 2: Create the IAM Role, With a Wildcard Trust Policy

This is the one step that makes everything else possible. Normally, a trust policy names one exact Kubernetes service account. Instead, we write it to match a pattern, using a wildcard (`*`).

**First, register the cluster's OIDC provider.** Before AWS can trust ID tokens issued by your EKS cluster, you have to tell IAM that this cluster's token issuer is legitimate:

```bash
aws eks describe-cluster --name <your-cluster-name> --region <your-region> \
  --query "cluster.identity.oidc.issuer" --output text
```

This prints a URL like:
```
https://oidc.eks.eu-north-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXXX
```

**IAM Console → Identity providers → Add provider**
- Provider type: OpenID Connect
- Provider URL: paste the URL above
- Audience: `sts.amazonaws.com`

**Now create the role. IAM Console → Roles → Create role → Custom trust policy**, paste (replacing the account ID and OIDC URL with your own):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.eu-north-1.amazonaws.com/id/<your-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.eks.eu-north-1.amazonaws.com/id/<your-id>:sub": "system:serviceaccount:loft-default-v-*:vc-*",
          "oidc.eks.eu-north-1.amazonaws.com/id/<your-id>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Read that condition carefully, this is the whole trick:

- `loft-default-v-*` matches any namespace that starts with this prefix. Every tenant cluster vCluster creates gets its own namespace following this exact pattern automatically, with the tenant cluster's name filled in after the dash.
- `vc-*` matches any service account starting with `vc-`. Every tenant cluster's control plane automatically runs under a service account named `vc-<tenant-cluster-name>`.

So instead of naming one tenant cluster, this trust policy says: trust any tenant cluster, in this project, no matter its name, as long as it follows the standard naming pattern vCluster always uses.

If you have multiple projects, you don't need to list them one by one either. You can widen the wildcard to cover all projects at once:

```json
"...:sub": "system:serviceaccount:loft-*-v-*:vc-*"
```

The only trade-off is that this now trusts any tenant cluster in any project, which is fine if all your projects share the same bucket and permissions, but worth keeping in mind if you need stricter, per-project isolation.

Attach the `snapshot-s3-access-policy` policy from Step 1 as the role's permissions. Name the role `snapshot-s3-irsa-role`.

A small honest note: when you paste this policy, the AWS console may show an advisory warning that the wildcard is "overly permissive," suggesting at least 6 fixed characters before a wildcard. This is just a security best-practice suggestion, it does not block you from creating the role, and in our tests the role worked correctly despite the warning.

### Step 3: Install vCluster Platform

```bash
helm upgrade --install loft loft \
  --repo https://charts.loft.sh \
  --namespace vcluster-platform \
  --create-namespace \
  --values ./platform.yaml \
  --wait
```

(`platform.yaml` contains your Enterprise license token.)

Get the admin login password:

```bash
kubectl get secret loft-user-secret-admin -n vcluster-platform -o jsonpath='{.data.password}' | base64 -d
```

Access the dashboard:

```bash
kubectl port-forward -n vcluster-platform svc/loft 9898:443
```

Then open `https://localhost:9898` in your browser and log in as `admin` with the password above.

### Step 4: Create a Tenant Cluster With the Role Annotation

Now create a tenant cluster (through the vCluster Platform UI, or the CLI), and in its configuration file, add this:

```yaml
controlPlane:
  advanced:
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>:role/snapshot-s3-irsa-role"
```

That's it. This one annotation is the only thing that links this specific tenant cluster to the shared role. Nothing else needs to be created in AWS for this tenant cluster specifically.

Add a scheduled snapshot configuration in the same file:

```yaml
snapshots:
  auto:
    schedule: "*/2 * * * *"
    retention:
      period: 30
      maxSnapshots: 30
    storage:
      type: s3
      s3:
        url: "s3://your-bucket-name/vc-test-c-snapshots"
```

(The schedule above runs every 2 minutes, which is convenient for testing. In a real environment you'd use something much less frequent, like every 12 hours.)

Here is the full `vcluster.yaml` for this tenant cluster, combining everything from this step:

```yaml
sync:
  toHost:
    ingresses:
      enabled: true
controlPlane:
  coredns:
    enabled: true
    embedded: true
  backingStore:
    etcd:
      embedded:
        enabled: true
  advanced:
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>:role/snapshot-s3-irsa-role"
snapshots:
  auto:
    schedule: "*/2 * * * *"
    retention:
      period: 30
      maxSnapshots: 30
    storage:
      type: s3
      s3:
        url: "s3://your-bucket-name/vc-test-c-snapshots"
```

### Step 5: Proving It Works

Wait about 2 minutes, then confirm the snapshot was created, either from the command line:

```bash
aws s3 ls s3://your-bucket-name/vc-test-c-snapshots/ --recursive
```

or from the vCluster Platform UI, under the tenant cluster's **Snapshots** tab.

If you repeat this for a new tenant cluster, using the same role annotation and a different snapshot path, it works the same way immediately, with no new AWS-side step for that new tenant cluster.

---

## Closing Thought

The most important lesson from this lab isn't really about vCluster or even AWS specifically. It's a general pattern worth remembering: whenever a system asks "which exact thing is allowed to do X?", check whether it also accepts a pattern instead of a single exact answer. That one design choice, an exact match versus a wildcard match, is often the entire difference between "works once" and "scales automatically forever."
