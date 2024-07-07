---
layout: post
title:  "Moving AWS Accounts and OUs Within An Organization - Not So Simple!"
date:   2024-07-05 01:00:00 -0500
image: assets/img/move-aws-account.png
tags: dev
author: fuller
description: Operational and security considerations when moving AWS accounts and OUs within an Organization
excerpt_separator: <!--more-->
---

![Moving an AWS Account](/assets/img/move-aws-account.png "Moving an AWS Account")

# Background

AWS Organizations enables the management and organization of AWS accounts via a hierarchy of organizational units (OUs). OUs are essentially directories in a tree structure and accounts can reside in OUs up to five nested levels deep. These OUs also allow for policies (Service Control Policies - SCPs) to be applied to groups of accounts under the OU tree.

<!--more-->

As an example, a company using AWS Organizations may group their accounts under `dev`, `stage`, and `prod` OUs and use those groupings to apply different policies based on the environment type. Alternatively, OUs could reflect the organization of the business, such as "engineering," "security," "finance," and "product" groups. These concepts can even be combined; an OU tree could look like: `Root > Product > Staging > Account A`.

However they are structured, OUs are not static. They can be renamed, moved, and deleted according to the needs of the business. So to can the accounts under those OUs. There are many reasons why an OU or account may need to be moved, but some common examples include:
* The account's purpose has changed
* The organization of the company changed, and accounts may need to be moved to accommodate such a re-org
* The organization matures and begins creating more granular hierarchy for their accounts
* Acquisitions (inbound or outbound) of parts of the business may require its infrastructure to be migrated elsewhere
* Security considerations, such as creating tighter boundaries between OUs using policies
* Cosmetic reasons, such as restructuring the OU hierarchy to make more sense to developers managing it
* As part of a pipeline when provisioning or deprovisioning accounts

Note: this post covers moving AWS accounts or OUs _within the same Organization_. For considerations when moving accounts to new Organizations, Houston Hopkins has written a good guide [here](https://gist.github.com/houey/fa1129edb2214f1d278010578ea29c18).

# Risks Ahead

Moving an AWS account or OU can be done via a single [API call](https://docs.aws.amazon.com/cli/latest/reference/organizations/move-account.html) and works by simply modifying the `parentId`. Every account and OU has a parent, linking it back to the root of the organization.

While the API call is straightforward, and can easily be reverted, the reality is that there are many operational and security implications to consider. In the sections below, we'll cover some of the risks, and ways to prevent unintended complications during these moves.

# Considerations

## Policy Attachments and Inheritance

Perhaps the most direct implication of moving an account or OU is that its parent path will change, impacting the inheritance of policies attached to those parents.

For instance, consider the following change:

Original Path:
```
Root > Product > Staging > Account A
```

New Path:
```
Root > Product > Production > Account A
```

`Account A` has been moved from the `Staging` OU to the `Production` OU. As soon as this API call completes, the following changes will take effect:

* Policies applied to `Root` will continue to apply
* Policies applied to the `Product` OU will continue to apply
* Policies applied to the `Staging` OU will no longer apply
* Policies applied to the `Production` OU will begin to apply

This may seem simple, but it's an important consideration because SCPs are global policies that have wide-ranging impact to the services running within the accounts to which they are applied. If an application in the `Staging` OU relies on `s3:GetObject` calls, and the `Production` OU has a `Deny` statement blocking those calls, then the application will begin experiencing `AccessDenied` errors.

## Policy References

Aside from policies that are attached directly to the account, OU, or its parents, policies in other places can reference OU paths as a means of conditionally allowing or denying access. These include IAM policies, trust relationships, S3 bucket policies, VPC endpoint policies, and other resource policies.

Consider the following IAM trust relationship policy, attached to an application's IAM role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "sts:AssumeRole",
            "Condition":{
                "ForAnyValue:StringLike":{
                    "aws:PrincipalOrgPaths":["o-myorganization/*/ou-staging/"]
                }
            }
        }
   ]
}
```

This trust relationship allows any entity from any account under the `ou-staging` OU to assume it. If part of your application workflow involves assuming this role, and the account performing the role assumption is moved from `ou-staging` to `ou-production`, then the condition will no longer pass and the assume role call will begin failing.

From a security standpoint, this policy also presents a risk because it delegates the enforcement of which entities can assume the role solely to a mutable property of their organization. Any change to the OU structure, accidental or malicious, risks the unintended side effect of creating a new access path.

### Condition Keys

There are several condition keys that can lead to the issue described above:

* [aws:PrincipalOrgPaths](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-principalorgpaths) - refers to the Organization path of the _principal making the request_
* [aws:ResourceOrgPaths](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-resourceorgpaths) - refers to the Organization path of the _resource targeted by the request_
* [aws:SourceOrgPaths](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sourceorgpaths) - refers to the Organization path of the _resource making a service-to-service request_ (only when made by an AWS service principal)

### A Note on Global Uniqueness

AWS guarantees that AWS Organization IDs (and their corresponding ARNs) are globally unique. However, OU IDs do not have the same guarantee (they are only unique within the same organization). When referencing an OU via the above condition keys, AWS recommends including the full path, including the Organization ID, to prevent a situation in which principals in an OU outside of your Organization are inadvertently trusted.

For example:

Policy allowing any access from principals in any OU with ID `ou-ab12-22222222`:
```
"Condition" : { "ForAnyValue:StringLike" : {
     "aws:PrincipalOrgPaths":["*/ou-ab12-22222222/*"]
}}
```
vs

Policy allowing any access from principals in the OU with ID `ou-ab12-22222222` in Organization `o-a1b2c3d4e5`:
```
"Condition" : { "ForAnyValue:StringLike" : {
     "aws:PrincipalOrgPaths":["o-a1b2c3d4e5/r-ab12/ou-ab12-11111111/ou-ab12-22222222/*"]
}}
```

## CloudFormation StackSets

In an AWS Organization, CloudFormation stacks can be centrally deployed across multiple member accounts from the management account using the CloudFormation StackSets feature. The stack configuration controls which AWS accounts the stack is deployed in via "targets," which operate based on account or Organizational Unit ID.

The [DeploymentTargets](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_DeploymentTargets.html) property of the CloudFormation StackSet can be set to any combination of account IDs and OU IDs. When an account is targeted, either directly via its ID, or indirectly via one of its parent OUs, the stack may be deployed to that account.

The exact behavior of a stack deployment is determined by a combination of the stack's [automatic deployment and retention settings](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-deployment.html):

* A stack with **auto-deployment** enabled will deploy to AWS accounts that are added to an OU targeted by the stack
* A stack with **auto-deployment** disabled will not deploy to AWS accounts that are added to an OU targeted by the stack
* A stack with **retention** enabled will not be deleted when the account is moved out of the OU
* A stack with **retention** disabled will be deleted when the account is moved out of the OU

These options can be configured on a per-stack basis. However, before moving an account, it's important to note which stacks will be deleted, retained, or deployed in the account as a result of its new OU path.

## RAM Shares

AWS [Resource Access Manager](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html) (RAM) enables the secure sharing of resources across AWS accounts. It works by creating "resource shares" that expose groups of resources in an owning account with consuming account(s) via a policy. This policy can target consumers by their account IDs, or by their OU IDs.

In their [documentation](https://docs.aws.amazon.com/ram/latest/userguide/working-with-sharing-create.html), AWS states:

> If the sharing is between accounts or principals that are part of an organization, then any changes to organization membership dynamically affect access to the resource share.

This is a critically-important concept; moving an AWS account to a new OU can immediately impact the set of resources it has access to via RAM.

Before moving an account, you can [review the set of resources shared with the account](https://docs.aws.amazon.com/ram/latest/userguide/working-with-shared-view-rs.html). For each resource, you should evaluate how the share was created. Resources shared directly with the account ID will not be impacted, while resources shared with the account by nature of its OU inheritance may.

## Control Tower Enrollments

AWS Control Tower is a governance platform that can help manage and provision accounts in a multi-account environment and enforce specific governance controls within those accounts. Existing accounts can be added to or removed from Control Tower by enrolling/unenrolling the account directly, or by [registering an existing](https://docs.aws.amazon.com/controltower/latest/userguide/importing-existing.html) OU from the Organization.

If you use Control Tower, and move an AWS account or OU under an OU that is enrolled in Control Tower, then that account, or all accounts under that OU, will be enrolled. This could have unexpected side effects, such as enabling governance controls that could impact running applications. There is some nuance to this, including differences in how preventative, detective, and proactive controls are enforced, so be sure to read through the [docs](https://docs.aws.amazon.com/controltower/latest/userguide/nested-ous.html#nested-ous-and-controls) for your specific configuration.

## Other Challenges

AWS does not provide a "dry run" or "plan" API for evaluating the potential impacts of moving an account within an Organization. The changes described above take effect immediately, so moving an account could result in instant `AccessDenied` errors as a result of newly applied policies, lost RAM shares, or no-longer-applicable trust relationships.

This challenge is made more difficult by the fact that these policies can exist in numerous places: the account being moved, SCPs in the Organization management account, trust relationships, RAM shares, and resource policies in any of the hundreds or thousands of other AWS accounts in the same Organization.

# Change Management Strategies

Changing an account or OU parent ID is straightforward; a single configuration option can be modified via API or deployed via infrastructure-as-code rollouts (e.g., Terraform's `aws_organizations_account` [parent_id](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/organizations_account#parent_id)).

However, for operationally-sensitive accounts, the challenges described above require a bit more investment in operational risk management. There are some steps developers can take to reduce the risk of these changes:

* If possible, consider temporarily applying the same SCPs attached to the new OU to the original OU or account to observe potential impacts to the application in a controlled environment
* While not ideal, you can also consider temporarily moving the account or OU for increasing periods of time while monitoring for `AccessDenied` errors in CloudTrail. For example, you could update the `parentId` to the new OU for 15 seconds, revert, monitor, update again for 5 minutes, etc. based on your risk tolerance.
    * Note: this strategy may result in in-scope CloudFormation StackSets being deployed and deleted in rapid succession which could cause other unintended side effects.
* Consider using the checklist below to manually evaluate the impact of an account/OU move.

## Monitoring for Errors

Following an account or OU move, errors will most likely appear in the following places:

* CloudTrail - `AccessDenied` errors referencing SCPs, with a message like `due to an explicit deny in a service control policy`
* Application Logs - access denied errors while attempting to access AWS resources controlled by resource policies (e.g., S3 buckets)
* CloudFormation StackSet Logs - monitor the "Deployments" tab of any in-scope stack sets to ensure the stacks are deployed or deleted properly

## Reverting
Reverting the move involves simply changing the `parentId` back to the original parent. The majority of the above policy considerations should instantly revert back to how they were applied prior to the move. However, CloudFormation StackSets may need to be reviewed for possible cleanup, depending on their auto-deploy and retention settings.

### A Warning About StackSets with IAM Roles

Many organizations leverage StackSets to deploy shared resources, such as IAM roles, used by monitoring or security tools. Its important to note that when a role is deleted and recreated, some AWS services, such as KMS, may no longer trust the new role, even if it has the same name and ARN (the policy, depending on how it is written, may reference the now-deleted role via an `AROA` value). This is a security feature, but can cause complications when recovering stacks that have been deleted due to an account move. You can read more about this edge case [here](https://www.lastweekinaws.com/blog/the-sneaky-weakness-behind-aws-managed-kms-keys/).

# Checklist

The checklist below, while not exhaustive, contains a set of checks that can be performed prior to moving an AWS account or OU:

- [ ] Which SCPs are attached to the account or OU and its _original_ parents, up to the root?
- [ ] Which SCPs are attached to the _new_ OU and its parents?
- [ ] Will this move result in any new policies applying to the account due to any deltas in these policy attachments?
- [ ] Will any CloudFormation StackSets targeting the account via its original parent OUs be deleted?
    - [ ] Will any deleted CloudFormation StackSets delete shared resources (such as IAM roles) that may need to be recovered?
- [ ] Will any CloudFormation StackSets targeting the account via its new parent OUs be deployed?
- [ ] Do any IAM policies, trust relationships, etc., in any AWS account within the Organization, target by OU? Does this OU include any of the original or new OU paths?
    - [ ] Grep your policies for the use of OU condition keys, including: `aws:PrincipalOrgPaths`, `aws:ResourceOrgPaths`, and `aws:SourceOrgPaths`
- [ ] Do any RAM shares share resources based on OU? Does this OU include any of the original or new OU paths?
- [ ] Will moving the account or OU trigger any Control Tower enrollments that could begin enforcing governance controls on the account(s)?

----

_This post was published by [wut.dev](https://wut.dev), a platform for managing AWS Organizations and Policies_.