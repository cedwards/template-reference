# Advancing your Security with Permission Boundaries

AWS’ Identity and Access Management (IAM) enables you to manage access for all identities & resources in a fine-grained and simple way. Besides, the well-known identity and resource-based policies, AWS offers you the advanced feature of [Permission Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) for restricting the maximum permissions of identities.

Overview of what we’ll talk about:

 - Quick introduction: identity & resources-based policies
 - Restricting effective permissions via boundaries
 - Use-case examples
 - Key takeaways

## Identity & resource-based policies

AWS distinguishes between different types of policies. Let’s dive into the most common ones real quick.

On the one hand, there are [identity-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_id-based) that are attached to a user, group, or role. The policy documents control which actions can be taken by the identity of which resources. Those can be separated into managed policies — standalone policies, attachable to multiple users, groups or roles — and inline policies — policies with a strict attachment to a single user, group or role. In our example, we’re adding S3 permissions for a Lambda execution role.

```
resource "aws_iam_role" "main" {
  name               = "lambda-exec-role"
  assume_role_policy = data.aws_iam_policy_document.assume.json
}

resource "aws_iam_role_policy_attachment" "main" {
  policy_arn = aws_iam_policy.main.arn
  role       = aws_iam_role.main.name
}

resource "aws_iam_policy" "main" {
  name   = aws_iam_role.main.name
  policy = data.aws_iam_policy_document.main.json
}

data "aws_iam_policy_document" "main" {
  statement {
    effect  = "Allow"
    actions = ["s3:PutObject", "s3:PutObjectAcl"]
    resources = [
      "arn:aws:s3:::backup-bucket/*",
    ]
  }
}
```

On the other hand, there are [resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_resource-based) that define how a certain principal (e.g. an AWS service) is allowed to access a resource. As the name expects, those policies can be attached to resources, e.g. an S3 bucket. In the following example, we’re granting the role `backup` of account `1234567890` access to our bucket-keeping backups.

```
data "aws_iam_policy_document" "main" {
  statement {
    effect  = "Allow"
    actions = ["s3:PutObject", "s3:PutObjectAcl"]
    principals {
      type        = "AWS"
      identifiers = [arn:aws:iam::1234567890:role/backup]
    }
    resources = ["arn:aws:s3:::backup-bucket/*"]
  }
}
```

As we’ve covered the basics, let’s get to the interesting part.

### Restricting effective permissions via boundaries

As mentioned in the beginning, boundaries are limiting the maximum permissions. They are *not* providing permissions on their own, but only restricting already granted permissions. So if we need access to S3, we need to explicitly give this permission in our identity or resource-based permission, even if our permission boundary allows this action.

> 💡 Permission boundaries are not limiting resource-based polices: created boundaries are only able to restrict permissions which are granted to an user by it identity-based policy. Resource-based policies are always granting additional permissions, regardless what’s inside the permission boundary. Even if it’s an explicit deny.

We can summarize the whole concept in a single diagram:

![policy venn diagram](https://miro.medium.com/max/612/1*STwRUqIm9DWUhWgCtKr_gw.png "Effective permissions based on identity-based, resource-based & boundary policies")

To attach a boundary to a specific role in Terraform, you need to provide it via `permissions_boundary`. Let’s extend our Lambda execution role from our introduction:

```
resource "aws_iam_role" "lambda" {
  name               = "lambda-exec-role"
  assume_role_policy = data.aws_iam_policy_document.assume.json
  permissions_boundary = aws_iam_policy.boundary_policy.json
}
resource "aws_iam_policy" "boundary_policy" {
  name        = "iam-boundary-policy"
  policy      = data.aws_iam_policy_document.boundary_policy.json
}
data "aws_iam_policy_document" "boundary_policy" {

  statement {
    sid    = "S3Access"
    effect = "Allow"
    actions = [
      "s3:*"
    ]
    resources = [
      "arn:aws:s3:::first-*"
    ]
  }
}
```

We’re now making sure that only S3 buckets with our expected prefix first- can be accessed. And that’s basically it! You can now create your boundaries based on different conditions and apply those to all users, groups and roles in your account.

> 💡 Additional note: if you really want to put effort into your boundary permissions, you can easily exceed the maximum policy size at AWS! So don’t be too specific if you’re making use of a lot of different services. Also, it’s not possible to attach multiple boundary policies to a single role!

### Use-case examples

There are a lot of really good use-cases for permission boundaries. For example, if we’re having multiple, completely independent applications within our AWS account and we want to separate them securely in a programmatic way.

 - We define that all resources related to our applications will be prefixed with their application name, e.g. application first will have resource names starting with first-, application second will have names starting with second- and so on.
 - We’ll also tag accordingly, because for some resources we can’t attach specific names and therefore can only control access based on attached tags, e.g. CloudFront distributions or certificates in ACM.
 - Now we can create our permissions boundary policy based on our known prefix and/or expected tag and attach this policy as permissions_boundary to all of our roles. Ascertain actions require unrestricted resource access via `*`, e.g. `dynamodb:ListTables`, make sure to have this covered in your boundaries.
 - If we want to use CodeBuild/CodePipeline as our Contentious Integration & Delivery solution, we can also attach our boundary policy to the executive roles of both services as a “normal” policy. With this, we’re granting all of our permissions directly, and also be sure we’re not breaching any boundaries.

## Key takeaways

Permission boundaries are an awesome way to enhance your account security by settings the maximum viable range of actions. With them, it’s much less likely that you’re granting unnecessary rights, as your having centrally managed boundaries that you can apply to all your users, roles, and groups.

Even if not all of your team is putting great effort into respecting the least privilege, you're still having certain limits covered easily. 🎉
