---
title: "Introducing Terraform Stripe Provider"
date: 2019-02-25 18:32:33
comments: true
slug: "introducing-terraform-stripe-provider"
tags:
  - go
  - open-source
  - infrastructure
---

I build products regularly, most of them don't survive their prototype phases.  In 2018, I built:

  * a (fast) FaaS with v8 and mruby
  * a cryptocurrency exchange prototype
  * a programmable cryptocurrency trading platform
  * some more stealth projects...

(As an aside, my Open Source work isn't included in this list as I considered it as a "horizontal" supporting these projects, but I'm more and more seduced by the idea of Open Source as a lifestyle business way of living, which I will try to explore in 2019.)

I often go as far as setting up Stripe integrations to get the pricing plans in there, but I felt it was too tedious to:

  * Create an account for that new prototype
  * Set up the prices in a spreadsheet, and reflect them there
  * Keeping them aligned with my app's code and Stripe

So in order to automate Stripe's setup I created a Terraform provider for Stripe. Billing as Code is great!

<!--more-->

### How does it work?

What's Terraform?

> Terraform is a tool for building, changing, and versioning infrastructure
> safely and efficiently. Terraform can manage existing and popular service
> providers as well as custom in-house solutions.

Terraform is a stateful tool that can operate changes on remote resources,
and reflect remote state changes in its data store.  Changing a resource
(adding one, removing one, updating one) should be done in one of its
configuration file, and Terraform will be able to generate a `plan` for you
(what it will do,) and run the plan for you (operate the changes on the
remote target.)  Terraform has plenty of providers, which implement the logic that will run behind the scene to operate these remote changes.  [AWS][aws], [GCP][gcp], [Heroku][heroku], and many more provide support for their services so people can track their configurations in their own repositories instead of relying on UIs.

What will be required to set it up is:

  * The actual binary of terraform-stripe-provider (requires a Go development environment for now)
  * A configuration file

Here's an example configuration file:

```
# stripe-configuration.hcl
provider "stripe" {
  api_token = "xyz"
}

resource "stripe_product" "my_product" {
  name = "My Product"
  type = "service"
}

resource "stripe_plan" "my_product_plan1" {
  product  = "${stripe_product.my_product.id}"
  amount   = 12345 # in cents
  interval = "month"
  currency = "usd"
}
```

A simple

```
terraform init; terraform apply
```

would create the `My Product` product and its associated plan for you.

Beware: the `tfstate` file it creates must be kept secure! It contains your
state, and some sensitive information (like passwords, tokens, and other ...
secure data.)


### What's supported?

All these resources are supported now:

  * Products
  * Plans
  * Webhooks Endpoints
  * Coupons


### How to get it

Please take a look at [the GitHub repo][terraform-stripe-provider], it
provides more examples and documentations on the supported resources, and
please give feedbacks!

I was fortunate enough to have amazing engineers from Stripe helping me out, even fixing some misbehavings APIs!  Thanks folks! <3


[terraform-stripe-provider]: https://github.com/franckverrot/terraform-provider-stripe