---
title: "How not to terraform - AWS security groups"
date: 2022-02-25
author: 
  name: Bevan Bennett
categories: blog terraform aws howto hownotto
tags: ['terraform', 'hownotto', 'iac']
---

<h2> {{ post.title }} </h2>
Being an exploration of the many ways to manage AWS security groups using terraform and some of the corresponding gotchas you are likely to encounter.
<hr>

<h4> Our scenario </h4>
For this discussion, imagine the following scenario.
We have a webservice in an AWS VPC that we want to allow restricted access to. This webservice is part of production infrastructure and needs to be accessible from other systems in the following subnets:
- 10.1.0.0/24
- 10.1.1.0/24
- 10.1.2.0/24

A few months later, we will be asked to remove access from 10.1.1.0/24 after that subnet is deprecated for unforseen reasons.
Let's see how some different terraform code approaches fare with this task...

<h4> Method the First - Inline </h4>
Checking the docs, the "aws_security_group" [^1] resource supports an inline syntax for rules, which seems straightforward enough. Let's give that a try.

```terraform
locals {
  webservice_cidrs = ["10.1.0.0/24", "10.1.1.0/24", "10.1.2.0/24"]
}

resource "aws_security_group" "webservice_access" {
  name        = "webservice_access"
  description = "Allow secure access to our webservice"
  vpc_id      = module.vpc.vpcid

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = local.webservice_cidrs
  }
}
```

Hey! It works! Unfortunately, it feels less than ideal.

Primarily, this approach means that we cannot add any rules except through this resource, because aws_security_group ingress blocks are incompatible with separate aws_security_group_rule resources. This is often a very useful thing to be able to do.

We end up using separate rule resources often in our terraform because they allow for better logical separation between modules that create services and modules that depend on those modules. For example, we might have a dbserver module that creates and exports its security group, then modules for services that need to access the db pass in the security group id as a module variable and add their own rule(s) to it specific for their security group or IPs. This means that adding or modifying the dependent services module does not affect the db module itself, and removing that dependent module also naturally cleans up its db access rules. This gets to be especially important when implementing cross-account access.
There's another potential issue here as well, but we'll get to that shortly...

<h4> Method the Second - aws_security_group_rule </h4>
Let's try using the dedicated aws_security_group_rule [^2] resources.

```terraform
locals {
  webservice_cidrs = ["10.1.0.0/24", "10.1.1.0/24", "10.1.2.0/24"]
}

resource "aws_security_group" "webservice_access" {
  name        = "webservice_access"
  description = "Allow secure access to our webservice"
  vpc_id      = module.vpc.vpcid
}

resource "aws_security_group_rule" "webservice_443" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = local.webservice_cidrs
  security_group_id = aws_security_group.webservice_access.id
}
```
Nice! Now we can manage rules separately from the security group itself.
That said, if this is used for production systems there's still going to be an issue, which we won't see until we need to make changes to this rule. Let's move forward to where we need to remove that subnet.

```terraform
locals {
  webservice_cidrs = ["10.1.1.0/24", "10.1.2.0/24"]
}
```
Seems ok. Plan unsurprisingly says:
```terraform
  # aws_security_group_rule.webservice_443 must be replaced
-/+ resource "aws_security_group_rule" "webservice_443" {
      ~ cidr_blocks              = [ # forces replacement
          - "10.1.0.0/24",
            "10.1.1.0/24",
            # (1 unchanged element hidden)
        ]
      ~ id                       = "sgrule-550362072" -> (known after apply)
      - ipv6_cidr_blocks         = [] -> null
      - prefix_list_ids          = [] -> null
      + source_security_group_id = (known after apply)
        # (6 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```
...
But let's look at what happens when we apply:
```shell
aws_security_group_rule.webservice_443: Destroying... [id=sgrule-550362072]
aws_security_group_rule.webservice_443: Destruction complete after 0s
aws_security_group_rule.webservice_443: Creating...
aws_security_group_rule.webservice_443: Creation complete after 2s [id=sgrule-1022565893]
```

Uh-oh... that "must be replaced" meant that terraform removed ALL the rules from this resource and then took a few seconds to put them back. That doesn't sound too bad, but if you're a large site getting several (or several thousand) hits per second, you just had 2s of downtime even for the CIDRs you didn't touch. There go your "nines".

Unless we're feeling super-cool and want to go re-write the terraform aws provider to try to be more clever about that, this isn't going to work. The previous approach also had the same issue, as we alluded earlier.

<h4> Method the Third - count the aws_security_group_rules </h4>
Ok, so we *could* split up that resource into multiple terraform resources, that should let us replace or remove one without impacting the others, right?
```terraform
resource "aws_security_group_rule" "webservice_443" {
  count             = length(local.webservice_cidrs)
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [element(local.webservice_cidrs,count.index)]
  security_group_id = aws_security_group.webservice_access.id
}
```
This makes planning slightly slower, but should give us the resilience we need, right?
Original apply looks good, let's see what happens when we remove that CIDR...
```
  # aws_security_group_rule.webservice_443[0] must be replaced
-/+ resource "aws_security_group_rule" "webservice_443" {
      ~ cidr_blocks              = [ # forces replacement
          - "10.1.0.0/24",
          + "10.1.1.0/24",
        ]
      ~ id                       = "sgrule-3406563736" -> (known after apply)
      - ipv6_cidr_blocks         = [] -> null
      - prefix_list_ids          = [] -> null
      + source_security_group_id = (known after apply)
        # (6 unchanged attributes hidden)
    }

  # aws_security_group_rule.webservice_443[1] must be replaced
-/+ resource "aws_security_group_rule" "webservice_443" {
      ~ cidr_blocks              = [ # forces replacement
          - "10.1.1.0/24",
          + "10.1.2.0/24",
        ]
      ~ id                       = "sgrule-1836778028" -> (known after apply)
      - ipv6_cidr_blocks         = [] -> null
      - prefix_list_ids          = [] -> null
      + source_security_group_id = (known after apply)
        # (6 unchanged attributes hidden)
    }

  # aws_security_group_rule.webservice_443[2] will be destroyed
  - resource "aws_security_group_rule" "webservice_443" {
      - cidr_blocks       = [
          - "10.1.2.0/24",
        ] -> null
      - from_port         = 443 -> null
      - id                = "sgrule-1553143985" -> null
      - ipv6_cidr_blocks  = [] -> null
      - prefix_list_ids   = [] -> null
      - protocol          = "tcp" -> null
      - security_group_id = "sg-06c87879a5b7143e4" -> null
      - self              = false -> null
      - to_port           = 443 -> null
      - type              = "ingress" -> null
    }

Plan: 2 to add, 0 to change, 3 to destroy.
```

Oh my. That effectively changed the index of every entry, which means that terraform now wants to, once again, delete and replace everything, which means unexpected downtime. Even worse, as there seem to be more API calls involved, it can take even longer than before to recreate the security groups.

We could address this with some clever `terraform state mv` commands to rename things, but that's a lot of extra work post-merge that needs to happen quickly and potentially across many many environments. It's non-trivial (or at least annoying) to do this rotation, as you need to add a temporary imaginary location.
```
terraform state mv aws_security_group_rule.webservice_443["0"] aws_security_group_rule.webservice_443["4"]
terraform state mv aws_security_group_rule.webservice_443["1"] aws_security_group_rule.webservice_443["0"]
terraform state mv aws_security_group_rule.webservice_443["2"] aws_security_group_rule.webservice_443["1"]
terraform state mv aws_security_group_rule.webservice_443["4"] aws_security_group_rule.webservice_443["2"]
```

Let's keep looking.

<h4> Method the Fourth - hard code everything? </h4>
We could just hardcode a separate aws_security_group_rule resource for every possible CIDR, but that cannot be used as a generic module, is incredibly suceptable to copypasta errors, and is hideous to look at. So we're not even going to go there.

<h4> Method the Fifth - The final form? for_each </h4>
So, is all lost? No! Thanks goodness the Hashicorp team has been listening and we've had one more option made available to us.
``` terraform
resource "aws_security_group_rule" "webservice_443" {
  for_each          = toset(local.webservice_cidrs)
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [each.value]
  security_group_id = aws_security_group.webservice_access.id
}
```

Last chance, let's see what happens when we remove that subnet this time.
```
  # aws_security_group_rule.webservice_443["10.1.0.0/24"] will be destroyed
  - resource "aws_security_group_rule" "webservice_443" {
      - cidr_blocks       = [
          - "10.1.0.0/24",
        ] -> null
      - from_port         = 443 -> null
      - id                = "sgrule-3406563736" -> null
      - ipv6_cidr_blocks  = [] -> null
      - prefix_list_ids   = [] -> null
      - protocol          = "tcp" -> null
      - security_group_id = "sg-06c87879a5b7143e4" -> null
      - self              = false -> null
      - to_port           = 443 -> null
      - type              = "ingress" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

```
Success!
While for_each may seem new and frightening and weird and unintuitive, it's also (for now) the only sensible option if you don't want to be surprised by unintended downtime or scaling issues down the line.

Thank you for staying with us through this very specific (but hopefully useful) adventure.

<hr>
[^1]: [Resource: aws_security_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)
[^2]: [Resource: aws_security_group_rule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule)
