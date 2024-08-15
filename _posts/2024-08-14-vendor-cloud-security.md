---
layout: post
title:  "Holding Cloud Vendors to a Higher Security Bar"
date:   2024-08-14 01:00:00 -0500
thumbnail: assets/img/vendor-security-bar.png
image: assets/img/vendor-security-bar.png
tags: dev
author: fuller
description: An opinionated, critical look at the state of security of vendor cloud integrations, along with recommendations for documenting and adhering to cloud security best practices for both vendors and customers.
excerpt_separator: <!--more-->
---

_An opinionated, critical look at the state of security of vendor cloud integrations, along with recommendations for documenting and adhering to cloud security best practices for both vendors and customers._

<!--more-->

<br/>

# A Position of Trust

The state of security of vendor cloud integrations is dreadful.

In my ~decade of working with cloud environments, I've seen all sides of the vendor/customer relationship; as a vendor, I've built products that integrate with customers' clouds, and as a customer I've integrated third-party vendors and SaaS products. In this time, I've been routinely surprised by the number of vendors who, despite their "we take security seriously" mantras, seem to have opaque, questionable, or even downright insecure, practices surrounding their integrations into customer cloud environments.

To state the obvious, having access to a customer's cloud environment, in any form, is a position of tremendous trust and responsibility. A key element of that responsibility is demonstrating a commitment to cloud security best practices and integration practices that will safeguard not only the data the customer has directly shared, but the rest of their environment as well.

Frankly, I believe the cloud vendor industry needs to be held to a higher bar. A plethora of insecure practices seem to still prevail, despite years of more secure alternatives, and I believe that it's up to customers to push back.

To be clear, both parties in the relationship bear responsibility:
* Customers are responsible for choosing the third-party vendors they integrate with (and reviewing the security of those integrations)
* Vendors are responsible for adhering to cloud security best practices when integrating with customer's cloud environments and providing safe configuration options for those customers

# No, I Won't Email You My Access Keys

Let's start with a simple example. If you, as a vendor, are asking your customers to generate AWS user access keys and secrets (and _worse_, send them to you over channels like email), you have _lost the game_. We could debate semantics and listen to your counterpoints about how you support regular key rotation, or "keep them safe with military grade encryption," but the point here is simple: a more secure, safer alternative (AWS cross-account IAM roles in this case) exists as an industry best practice. Asking me for keys tells me a few things:

* You're not running your infrastructure in AWS (which is fine) but want to connect to customers who are, and aren't willing to spin up your own account (or leverage options like [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html)) to facilitate the connectivity
* You don't have experience with these kinds of cloud security best practices
* You do, but decided to ignore them
* You do, but think you can provide your own safer alternative (do you roll your own crypto too?)

None of these are good! At a minimum, it forces me, the customer, to take on technical security debt. At worst, it gives a sneak preview into the _rest_ of your security practices, which likely follow a similar pattern. This is where the relationship should end[[1](#footnotes)]. Please come back when you support IAM roles.

# It Gets Worse

Cloud integrations can be quite complex, involving numerous managed services, so I won't attempt to enumerate every possible security faux pas. But to give some insight into the kinds of things I've seen:

* "We don't really have a diagram, or list, of cloud resources we use, or how we connect to your environment. But our sales team can walk you through our SOC 2 certification."
* "We need the `iam:AttachRolePolicy` permission because we use it to update our own permissions from time to time to facilitate new features."
* "We don't use external IDs [[2](#footnotes)] in our cross-account roles because it makes the onboarding experience more complex."
* "Deploy this CloudFormation template that configures our ECS containers using the `AdministratorAccess` policy"
* "We don't support IMDSv2 on the EC2 hosts we run in your account; you'll need to ensure v1 is enabled."

# Questionnaires Don't Help

Some enterprising security teams have attempted to get ahead of these problems by... throwing a bunch of questions in a spreadsheet and then asking vendors to fill them out. But here's a dirty (well-known at this point?) industry secret: the people responding to those spreadsheets are often 1) not the vendor's security team, 2) not even a technical team, and 3) incentivized to move the deal along as quickly as possible by clicking whatever the most positive response is. "How do you properly safeguard service secrets?" "Sure, sure, yeah we do that using industry best practices..."

Also, while some companies certainly try, listing every possible cloud security concern and misconfiguration in a spreadsheet is a colossal waste of time. There are 200+ services in AWS alone, each with their own integration options. That one misconfiguration hidden in tab 6, cell BZ3031 won't matter. Much has been written about the plague of security vendor questionnaires; I won't waste time beating a dead horse here.

# So What Should Vendors Do Instead?

Before I give off the wrong impression that _all vendors are scary; don't trust them_, I'll note that vendors play a critical role in extending the capabilities of customers' cloud environments. There are thousands of well-intentioned logging, monitoring, security, compliance, and other vendors that comprise a vital ecosystem.

With a few basic tweaks, which I'll outline below, vendors can not only improve their security model, but also exude additional security competency. Vendors should _want to do this_; it makes customers much more likely to trust them which translates to more sales (heck, you could probably argue for marketing budget to update processes!).

My plea to vendors is...

## Document, Document, Document

This isn't going to sound revolutionary, but I believe the simplest demonstration of security competency is to provide clear, authoritative documentation about the security practices of your integration. Don't force the customer's security team to open support tickets to find out this basic information, share it proactively. This documentation should include a few things:

* What does an integration with your service look like for customers?
    * Am I creating a new cloud account to host your application? Or deploying it into an existing environment?
    * What cloud services are you asking me to spin up? EC2? Lambda? Be detailed about why each is required.
    * What cloud regions are involved?
    * An infrastructure and data flow diagram goes a long way here!
* How does the installation work?
    * Are you providing a CloudFormation template? Script? Manual instructions?
    * If so, include links in your docs.
* Delineate the differences between your hosted (SaaS) or on-prem (self-hosted) models.
    * What am I running? What are you running? How do they securely talk to each other?
* What access am I granting the components of your application?
    * List the actual permissions, and why you need each one
* Who, or what, systems are getting access to my data? How?
* How are you practicing tenant isolation? What controls keep my data separate from your other customers?
* How are updates facilitated?
    * Do they happen automatically? Require a support engineer from your side?
* What cloud security practices are you following for your own infrastructure?
    * I don't need a comprehensive list of _everything_, but some detailed examples helps build confidence that you know what you're doing.
    * Note: be specific here. Generic phrases like "we use state of the art encryption" aren't helping.
* What happens after I'm done using your product? How do I disconnect it, delete it, remove my data, etc.?

These are just a few examples of the basics. Depending on your particular integration model, you might want to include more details about the services you use (e.g., sharing data on S3? Talk about S3 bucket policies. Using ECS? Dive a bit deeper into details about the VPC, network configurations, container configuration, etc.)

This documentation should ideally be public, but at the very least available _before_ a customer chooses to purchase your product (it's part of the sales process after all).

## Start Off on the Right Foot

The proverbial doorway to your product is the initial integration - how your product connects to a customer's cloud environment. Fortunately, most cloud providers have provided clear best practices for third-party connectivity. Unfortunately, many vendors decide to do their own thing instead.

### Initial Connectivity

Nothing makes me more impressed with (and likely to use) a vendor than one that simply says the following:

> Our application connects to your cloud via a third-party cross-account AWS IAM role. We generate a unique, external ID for use in the trust relationship and the IAM policy attached to the role has the following permissions...

When I read this, I know that you know (more or less) what you're doing. You'd be surprised how many vendors needlessly complicate, or completely botch, this step by:
* Asking for access keys and secrets.
* Using `*` in the trust relationship.
* Using `:root` in the trust relationship (use a specific IAM principal instead).
* Not using an external ID in the trust relationship (or allowing the customer to provide their own).
* Allowing their application to dynamically update the trust relationship.

One of the more common ways of provisioning this access is to supply customers with an infrastructure-as-code template (e.g., AWS CloudFormation). As an example, for AWS, the flow is quite straight-forward:

1. The customer begins onboarding the integration in the vendor application
1. The vendor application dynamically generates an external ID for connection
1. The customer launches the CloudFormation template, passing the external ID as a parameter
1. The template creates the IAM role, [referencing the external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) in the trust policy
1. The template outputs the ARN of the newly-created role, which the customer can then copy into the vendor application, completing the connection.
    1. An optional step is to include a CloudFormation custom resource that invokes a vendor API with the newly created role's ID instead. This is a good UX improvement, as the application's onboarding wizard can simply poll until the creation completes.
1. In the event that the role is required in _every_ AWS account, the above CloudFormation template should be usable in CloudFormation StackSets
    1. Most customers with moderately-large AWS environments will likely require this; manually deploying templates across 100s of accounts is not fun!
1. The vendor application's role then performs an `sts:AssumeRole` on the customer-provided role, completing the connection.

A good AWS IAM trust relationship looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:role/vendor-application"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
            "sts:ExternalId": "f4b50fc7-dbc8-4182-85d6-7ba29c1213e8"
        }
      }
    }
  ]
}
```

### Permissions

You (or your product team) likely want lots of access (although you shouldn't; it's not good for you either) and your customers want the least access. Can we find a happy medium? At a minimum, you need to differentiate between read-only access and write access, and describe the scope of those permissions.

In an ideal world, every vendor policy will contain the absolute minimum set of permissions required - `s3:GetObject` on `arn:aws:s3:::example-bucket`, instead of `s3:Get*` on `*`, for example. But some applications are complex, and make hundreds of different API calls on many different resources. Additionally, the application may be updated in the future with new functionality. So there's a balancing act.

Some rules of thumb I like to follow (examples using AWS IAM):
* Never use admin, or global access (`AdministratorAccess`, `*.*`, etc.) or similarly-broad policies (e.g., `PowerUserAccess`)
* Write permissions should be tightly scoped to both the action (e.g., `s3:CreateBucket`) and the resource (e.g., `arn:aws:s3:::example-bucket`)
* Read permissions should be as tightly scoped as possible (I'm slightly more lenient with `lambda:List*`, but only if you define a resource, or wildcard string, like `arn:aws:lambda:us-east-1:123456789101:function:vendor-name-*`, or really need to see all the Lambdas in my account)
* Permissions related to identity should be used _very_ sparingly, scoped to a resource, and accompanied by clear explanations of why they’re required
    * Ex: `iam:CreateRole`, `iam:CreatePolicyVersion`, `s3:PutBucketPolicy`, etc.
    * Review [this article](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/) about privilege escalation methods and make sure your policy doesn't include any of these.
* Be careful when using cloud provider managed policies
    * Ex: AWS's `ReadOnlyAccess` policy is actually much more permissive (granting access to all S3 bucket objects) than is likely required.

## Future Updates

Customers want features, and your application likely will need to be updated in the future; both sides understand this. What's important is to clearly document how the updates happen, who facilitates them, and what changes (to both the application functionality and the security model) are involved.

In my experience, customers simply want to be in control of how changes happen in their environment. Automatic updates are nice and simple, but they imply that the vendor can deploy (potentially untested or breaking) changes, and the vendor can introduce new vulnerabilities or security risks into the customer environment without review.

Again, cloud integrations are complex, so this is not a comprehensive list, but some examples of questions you should proactively address include:

* Was your application integrated via a CloudFormation template? You can provide versioned, updated template links that customers can roll out on their own. Be sure to include a diff and changelog.
* Is your application running as a container in the customer's environment? Provide a registry and tag they can use to mirror into their own internal registry of choice.
* Does your application pull updates from an endpoint? If so, provide customization options so customers can choose when, and where, these updates are pulled.

# And What Should Customers Do?

Customers turn to vendors to address a business requirement. The teams choosing the vendors are often not the security team evaluating the vendor's security practices; so the relationship might actually involve four (or more) parties: the customer team asking for the software, the customer team evaluating the software security, the vendor sales or success team(s) managing the relationship, and the vendor's internal security and infrastructure teams.

In the spirit of forging a good relationship, there are a few key things that customers can do to help facilitate the vendor relationship:

## Manage Communications

Vendors want your business, but it's frustrating for a vendor to receive emails from 8 different customer teams asking questions (often commingled) about security, compliance, sales/pricing, and product features. Whoever is managing the vendor relationship should manage these interactions; funnel your communications with the vendor through key contact points. This will help avoid cross-talk and inaccurate information. Don't make vendors sign up for multiple different vendor management platforms.

## Be Upfront About Your Security Requirements

Do all vendor applications get deployed to their own cloud account? Do you mandate vendors to re-attest to your annual security questionnaire? Are AWS access keys a deal breaker? Being upfront with your security requirements is mutually beneficial - you have the initial leverage as a potential customer, and the vendor has a clear understanding of what's required to properly support and price their product (avoid the "year two price jump" because you needed 200 unplanned hours of custom dev work in the first year).

If you really require the security vendor questionnaire, at least send one that's relevant. A vendor integrating with your cloud environment shouldn't need to copy and paste "N/A" to 150 questions about their datacenter practices.

## Provide Access to Engineers Where Needed

Engineer-hours are expensive, but what's more expensive is letting a team without detailed knowledge of your cloud infrastructure manage the vendor requirements during the purchasing phase and then finding out that the integration can't be supported because _insert technical reason here_.

If you have complex requirements, or suspect the integration will require custom security configurations, be upfront with the vendor and provide (time scoped) access to an engineer on your team who can accurately answer the vendor's questions.

## Document, Document, Document

Similar to my plea for vendor documentation, customer integration documentation can go a long way as well (and can also reduce the need to "just get a member of the security team on the phone so we can talk about these requirements...")

Some of this information may be sensitive, so of course use your discretion in how much to reveal, but here are some examples of things that customers can include in documentation sent to vendors:

* What cloud providers and regions are you using?
    * Are you willing to entertain a new one if, for example, a vendor uses AWS for their self-hosted options and you only use GCP?
* Roughly, what does your infrastructure look like?
    * What cloud services do you use? How do deployments work?
* How do you prefer to deploy vendor software?
    * Should they send you a script? Terraform template? CloudFormation?
* What does your vendor management process look like? What should the vendor expect?

While most vendors are not going to completely rewrite their integration to meet niche requirements (expensive enterprise agreements notwithstanding), having visibility into your security requirements early helps get the right people engaged early on both sides.

# Conclusion

Although vendors provide valuable services in the cloud ecosystem, they must take responsibility for properly securing not only their own cloud environments, but also the integrations through which they gain access to customers' environments. By holding a high bar in the clarity and substance of their security requirements, customers can also help ease this burden for vendors.


#### Footnotes

_[1] I am, of course, being dramatic for effect. Yes, there are business requirements, and security exists to enable the business while mitigating risk. This is an opportunity for security teams to surface those risks to leadership. If the business can't proceed without this vendor, and they only support keys, then you've said your piece; it's now your job to provide mitigating controls._

_[2] [AWS Confused Deputy Problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)_

----

_This post was published by [wut.dev](https://wut.dev), a client-side platform for viewing, managing, and debugging AWS security policies, controls, and access issues related to AWS Organization Service Control Policies (SCPs), IAM policies, and resource policies._