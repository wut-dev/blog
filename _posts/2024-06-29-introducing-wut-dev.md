---
layout: post
title:  "Introducing Wut.Dev"
date:   2024-06-29 01:00:00 -0500
image: assets/img/cover.png
tags: update
---

![Cover Image](/assets/img/cover.png "Cover")

As cloud infrastructure usage continues to grow, it has become common (recommended) practice to organize the accounts that contain that infrastructure by team, project, region, environment, and other categories. This has led to a rapid expansion in the number of cloud accounts that many companies, and even individuals, are managing. Some companies have hundreds of accounts, and it's not uncommon to have _thousands_ of accounts.

To help manage this sprawl, AWS long ago introduced "Organizations," as a service for its users. Organizations allows for the centralized management of AWS accounts. Accounts can be created, organized into logical groups ("units"), and deleted from a single management interface. Additionally, governance policies, called "Service Control Policies" (SCPs), can be applied to accounts, units, or the entire organization from the root.

While Organizations and SCPs have become commonplace for cloud and infrastructure security teams, their usage, along with the size of some cloud environments, present some unique challenges:

1. **Visibility**: AWS provides a basic UI for Organizations, but the majority of companies manage their Organization and accounts via infrastructure as code (e.g., Terraform, CloudFormation). While this is a best practice, parsing thousands of lines of Terraform code to figure out which OU an account exists in, and which SCPs affect it, can be difficult. Additionally, Organizations are a tree structure, but there is no AWS-provided visualization of that tree.
2. **Policy Inheritance**: SCPs can be applied to the Organization root, OUs, or directly to an account. Accounts inherit policies from their parents in the Organization tree, making the final allow/deny decision difficult to evaluate. In an Organization of hundreds of accounts, it can be nearly impossible to quickly determine the coverage of a particular policy across the set of all accounts (i.e. which accounts does a policy apply to?).
3. **AWS Limits**: AWS limits the number of nested OUs, the number of policy attachments, and the size of the policies themselves. In large Organizations, this often means that security teams are using creative measures to try to reduce policy size and limit overlapping or duplicate policies. There are few, if any, tools that make this simple.
4. **Developer Workflows**: In most companies, a security or central infrastructure team manages the AWS Organization while various development teams may interact with subsets of accounts within that Organization. This can make ownership and discoverabilty challenging. Developers may also struggle to understand which policies may be affecting their workflows (is it an SCP on the OU two levels up, or an unrelated IAM issue, for example). Security teams are often pulled into debugging issues because developers don't have the access to to view, or interpret, this information.

I started wut.dev as a way to address some of these issues. At its core, wut.dev is a visualization tool; you load your AWS Organization and get instant access to an interactive tree diagram, reflecting the OU and account structure, policies, and tags. The entire application is client-side; wut.dev has no API and all interaction with the AWS APIs is done via the JavaScript SDK in your browser. This means we never see your data.

Wut.dev also addresses the issues above by providing additional tools and visualizations, such as the "Policy Coverage Map" that shows a single table of all SCPs and all accounts with green/red "covered" or "not covered" indicators. Wut.dev is in active development, so you can expect more features soon.

Wut.dev is in beta, but you can get started today at [wut.dev](https://wut.dev). There is no sign in, no ads, no trackers, and no analytics.