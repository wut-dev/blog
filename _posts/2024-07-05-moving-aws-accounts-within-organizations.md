---
layout: post
title:  "Moving AWS Accounts Within An Organization"
date:   2024-07-05 01:00:00 -0500
image: assets/img/cover.png
tags: dev
---

![Cover Image](/assets/img/cover.png "Cover")

# Background

* AWS Organizations
* "move-account" API (https://docs.aws.amazon.com/cli/latest/reference/organizations/move-account.html)
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