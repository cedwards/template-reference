# Designing Least Privilege AWS IAM Policies for People

Designing least privilege IAM policies for people can be a tricky puzzle. Code is deterministic and the permissions it requires are unlikely to change between invocations. People, however, behave in unexpected ways, and may need powerful roles to do their job. Administrative permission can be abused by an attacker, or misused by an engineer who accidentally makes a manual change that bypasses automated testing and impacts the production system. Both scenarios could have a massive impact on security.

We can reduce the blast radius of everyday operations by limiting their ambient authority. This article describes a path towards reducing administrative role assumption by creating least privileged employee IAM access roles.

## Considerations when designing for least privilege

### Least privilege must be enough privilege

Well-crafted IAM is flexible in unexpected situations. Least privilege policies shouldn’t introduce errors during legitimate activity, like manual troubleshooting. As with any other change to engineering workflows, improvements to employee IAM policies should be rolled out with care. Access errors should be audited and resolved.

### Think about use cases

To model least privilege, understand how people behave. Some behavior can be inferred from logs, but engineering organizations are dynamic. Which teams are making production changes? What happens when the interns show up in the summer? Changes that limit access need to be designed with input from a diverse set of folks: product engineers, DevOps engineers, IT admins, people operations, support, and so on.

### Design for the future, not just for the past

Engineers might need sparingly used permissions to respond to incidents. People may want to use a new feature of an AWS service that is designed to be manually operated. There may be changes to the way that AWS IAM works. IAM roles will need to be monitored and maintained so that they can grow with the organization.

## Existing approaches to defining AWS least privilege

### Automated IAM policy generation from logs

In many cases, we can determine IAM least privilege policies based on historical audit logs. There have been many attempts to do so, including the recent AWS release of [IAM Access Analyzer policy generation](https://aws.amazon.com/blogs/security/iam-access-analyzer-makes-it-easier-to-implement-least-privilege-permissions-by-generating-iam-policies-based-on-access-activity/) in April 2021. Follow-on upgrades include the ability to [use organizational CloudTrail S3 buckets](https://aws.amazon.com/about-aws/whats-new/2021/08/iam-access-analyzer-generate-iam-policies/) and [support for more than 50 services](https://aws.amazon.com/about-aws/whats-new/2021/08/iam-access-analyzer-expands-services/) ([diff](https://www.diffchecker.com/1Osnaw0Z) by [Scott Piper](https://twitter.com/0xdabbad00)). This feature is promising for quickly creating least privilege roles, especially for roles used by code, although it is still rough around the edges. For example, configuring permissions to the CloudTrail logs bucket can be challenging, and there is a limit on the amount of logs that can be analyzed.

Access Analyzer policy generator is similar to an earlier open-source project [CloudTracker](https://github.com/duo-labs/cloudtracker), which shows which permissions are being used by a role based on ElasticSearch or Athena analysis of CloudTrail logs. These projects were also inspired by [RepoKid](https://github.com/Netflix/repokid), which screenscraped the older Access Analyzer UI (irony!) to construct least privilege IAM policies, although [later updates added a hook for CloudTrail](https://github.com/Netflix-Skunkworks/repokid-extras).

Using audit logs from CloudTrail to generate IAM policies isn’t always sufficient. [Travis McPeak](https://twitter.com/travismcpeak), an author of RepoKid, [wrote](https://ermetic.com/blog/cloud/least-privilege-automated-and-gift-wrapped-for-you/1) that “automatically building policies from CloudTrail isn’t a complete solution because many actions are still not logged, so policies generated this way may miss vital permissions.” For instance, data events for DynamoDB and CloudTrail aren’t logged to CloudTrail by default, and iam:PassRole [isn’t logged in CloudTrail](https://ermetic.com/blog/aws/auditing-passrole-a-problematic-privilege-escalation-permission/). Furthermore, CloudTrail names don’t reliably map to IAM names, leading to a variety of corner cases.

Another concern is that large policies are unsupported. The [AWS limit is 10,240 characters for role policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html), which they seem to bypass - the [much maligned ReadOnlyAccess](https://www.google.com/search?q=ReadOnlyAccess&oq=ReadOnlyAccess&aqs=chrome..69i57j69i59j69i60.118j0j1&sourceid=chrome&ie=UTF-8) weighs in at 43,536 characters at last measurement. Fine-grained policies for engineers who generate numerous CloudTrail logs accessing the AWS console, for instance, may exceed this limit.

## Automated IAM policy generation from DSL

Instead of using CloudTrail logs, you can automatically generate policies based on your own knowledge about what access is needed. [Policy Sentry](https://github.com/salesforce/policy_sentry) lets you generate IAM policies through a DSL (domain-specific language) based on YAML. I wrote a similar tool at Clever called [terrafam](https://github.com/Clever/terrafam).

**Example Policy Sentry YAML file**
```yaml
mode: crud
read:
 - 'arn:aws:ssm:us-east-1:123456789012:parameter/parameter'
write:
 - 'arn:aws:ssm:us-east-1:123456789012:parameter/parameter'
list:
 - 'arn:aws:ssm:us-east-1:123456789012:parameter/parameter'
tagging:
 - 'arn:aws:secretsmanager:us-east-1:123456789012:secret:mysecret'
permissions-management:
 - 'arn:aws:secretsmanager:us-east-1:123456789012:secret:mysecret'
```

The challenge of using a DSL is two-fold - one, each organization has a unique set of permissions that they want to express, and DSLs can be inflexible or too generic. Additionally, you still need to figure out what permissions users need.

## Refactoring an administrator employee role

For refactoring employee IAM role policies that have administrative access, you can combine an analysis of CloudTrail and automatic code generation of policies to achieve a tailored policy. Let’s take a look:

### Find out what the existing role is being used for

To determine what the role is currently being used for, you can analyze CloudTrail using Athena [set up with partitions](https://alsmola.medium.com/partitioning-cloudtrail-logs-in-athena-29add93ee070). If you wanted to understand what the IAM role has been accessing, you can use this rollup query:

```
SELECT eventsource, eventname FROM aws_cloudtrail WHERE userIdentity.arn LIKE '%<role name>%' GROUP BY eventsource, eventname
```

You’ll want to add partition filtering to the WHERE clause, like YEAR=2021 and MONTH=09. You may also want to filter out calls to services that aren’t really being used. For instance, visiting the AWS console landing page for a service will generate a flurry of CloudTrail logs that can clutter up a least privilege policy. If you’re thinking about compliance, you might want to filter against the [AWS Services In Scope](https://aws.amazon.com/compliance/services-in-scope/) for relevant programs, to make your least privilege policy usable as evidence in your audits.

## Define the roles that can replace it

Next, figure out the groupings of permissions that you can give you people at your organization. You can use IAM roles not only to group permissions, but to configure session expiration, auditing and alerting, and so on. A suggested three-tier strategy could like like:

## Viewer

 - Used frequently for validating configuration
 - Long expiration time (e.g. 24 hours) to encourage usage

## PowerUser

 - Can view and modify data and some infrastructure
 - Should be used somewhat frequently by engineers for one-off tasks
 - Should be easily auditable - quickly viewable in SIEM
 - Short expiration time (e.g. 1 hour) to discourage unnecessary usage

## Administrator

 - Can modify all data and infrastructure
 - Should be used very rarely by administrators, for example during incidents
 - Should cause alert when used

## Grant roles appropriate IAM policy

Next, come up with a strategy for assigning permissions to these roles in a way that maps intended use cases to IAM policy. Following the three-tier strategy, you could define:

## Viewer

 - Configuration read (Describe*, List*) for all services used in account

## PowerUser

 - Data read and write (Get*, Put*) for managed data stores like DynamoDB, S3
 - Additional policies for specific tasks

## Administrator

 - AdministrativeAccess managed policy

You can define IAM roles and policies in configuration as code tools like Terraform. Policies can also be generated by DSL-tools like Policy Sentry. You can also write custom scripts to generate policies. The benefit of using these tools is that changes are controlled through an approve/merge process, keeping the policies maintainable as they become more complex over time.

## Give the roles to the right people

If you want to determine who is using that role and you’ve configured the role session name to be the SAML user’s email address, you can list the users that have assumed the role this way:

```
SELECT count(*), userIdentity.uysername as email from aws_cloudtrail where eventName LIKE 'AssumeRole%' AND requestParameters like '%<role name>%' GROUP BY userIdentity.username
```

As you zoom out to figure out your organizational access control strategy, you need to make several non-technical process decisions. Who gets to define who has access to what? You may need to distribute responsibilities - administrators can be defined by the appropriate division Director, etc.

When engineers log into AWS, they will see a new set of roles. Perhaps some engineers will no longer be able to access an administrative role in production accounts, and won’t be able to directly modify production AWS infrastructure outside of documented use cases. Communicate these changes and ask for feedback to ensure you aren’t introducing friction in the development process. Consider ways to make the role changes improve the developer experience. For instance, you can centrally manage and push the AWS CLI role assumption tooling configuration (e.g. `./aws/config`, `./saml2aws`) to engineering laptops.

Role assignment doesn’t need to be permanent. There are a variety of tools, including Sym, Opal, ConductorOne, ConsoleMe and others that intermediate IAM role access requests and support approval workflows, self-service, and so on. You may want to sync your on call rotation to assure the oncall engineer has access to the Administrator role.

Least privilege is a popular policy, but it can be difficult to operationalize. As you tighten permissions for cloud engineers, consider how to keep your policy simple and flexible. Combine the auditable data with real-world observations, and use tools to help you create and update the policy. Figure out who will be affected by the changes, then communicate the new process to them. Make sure you communicate the administrator “break glass” strategy. Roll out to a friendly set of users first. Double check you’re monitoring for access denied issues and update policies as needed. Once you feel confident that you have provided the right set of permissions, roll it out to everyone. Now, you can do the fun part; deleting the old role.

Cover photo by [Ryoji Iwata](https://unsplash.com/@ryoji__iwata) on [Unsplash](https://unsplash.com).
