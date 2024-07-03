---
layout: post
title:  "Moving AWS Accounts Within An Organization"
date:   2024-07-05 01:00:00 -0500
image: assets/img/cover.png
tags: dev
---

![Cover Image](/assets/img/cover.png "Cover")

# Background

AWS Organizations enables the management and organization of AWS accounts via a heirarchy of organization units (OUs). OUs are essentially directories in a tree structure and accounts can reside in OUs up to five nested levels deep. These OUs also allow for policies (Service Control Policies - SCPs) to be applied to groups of accounts under the OU tree.

As an example, a company using AWS Organizations may group their accounts under "dev," "stage," and "prod" OUs and use those groupings to apply different policies based on the environment type. Alternatively, OUs could reflect the organization of the business, such as "engineering," "security," "finance," and "product" groups. These concepts can even be combined, an an OU tree could look like: `Root > Product > Staging > Account A`.

However they are structured, OUs are not static. They can be renamed, moved, and deleted according to the needs of the business. So to can the accounts under those OUs. There are many reasons why an OU or account may need to be moved, but some common examples include:
* The account purpose has changed
* The organization of the company may change, and accounts may need to be moved to accomodate such a re-org
* The organization matures and begins creating more granular heirarchy for their accounts
* Acquisitions (inbound or outbound) of parts of the business may require its infrastructure to be migrated elsewhere
* Security considerations, such as creating tighter boundaries between OUs using policies
* Cosmetic reasons, such as restructuring the OU heirarchy to make more sense to developers managing it

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

This may seem simple, but it's an important consideration because SCPs are global policies that have wide-ranging impact to the services running within the accounts to which they are applied. If an application in the `Staging` OU relies on `s3:GetObject` calls, and the `Production` OU has a `Deny` statement blocking those calls, then the application may begin experiencing `AccessDenied` errors.

## Policy References



## StackSets

## RAM Shares

# 

- AWS doesn't have a dry-run mode or "plan" capability
- Options/Strategies:
    - Remove all the SCPs first, then re-add them one by one
    - Quickly change the parent, monitor for errors, and then change back, gradually increasing the time
    - Apply the SCPs from the new OU path directly to the account being moved _first_ and watch for errors

- How to monitor for errors (CloudTrail)

# Checklist

- [ ] Which SCPs are attached to the account or OU and its original parents, up to the root?
- [ ] Which SCPs are attached to the new OU and its parents?
- [ ] Will this move result in any new policies applying to the account due to any deltas in these policy attachments?
- [ ] Will any CloudFormation StackSets targeting the account via its original parent OUs be deleted?
- [ ] Will any CloudFormation StackSets targeting the account via its new parent OUs be deployed?
- [ ] Do any IAM policies, ni anyt 


* AWS Organizations
* "move-account" API ()
    * Referring to _intra-Organization_ moves
    * For _inter-Organization_ moves, see: https://gist.github.com/houey/fa1129edb2214f1d278010578ea29c18
* Reasons to move accounts
* OU-based sharing (e.g., RAM)
* TODO: do any other controls/policies/etc. operate based on OU path references?

# Considerations

1. SCPs attached to the original parent OU tree
1. SCPs attached to the new OU tree
1. Policies (SCP, IAM, resource, etc.) that use "PrincipalOrgPaths" conditions
1. StackSets - targeting an OU - will they be auto-removed or auto-deployed? This could unexpectedly delete, or deploy, infra
1. RAM OU-based sharing
1. VPC endpoint policies
1. Control tower?
1. Identity center?
1. Cost/billing/limits?
1. Other
    1. Does AWS have any rate-limits or service limits based on OUs (even internally)?

# What to Monitor
1. Access denied errors spiking
1. S3 access logs
1. 

# Reverting
1. Easy to revert via the same "move-account" API call (just swap the parents)

https://docs.aws.amazon.com/ram/latest/userguide/scp.html