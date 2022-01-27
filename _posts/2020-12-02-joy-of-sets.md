---
layout: post
title: "Terraform - The Joy of Sets"
date: 2020-12-02
categories: terraform
github_comments_issueid: 5
---

One of the changes that comes with the new versions of Terraform (>0.11) is that resource outputs are now sets instead of lists and objects instead of maps. Presumably this makes it easier to code when you don't have to guarantee the order of values but it makes things harder for users - well, for me anyway.

Typical errors when trying to use sets as lists without any conversion:

```(terraform)
Error: Error in function call

  on test.tf line 42, in output "subnet_ids":
  42:   value = element(data.aws_subnet_ids.firewall.ids, 0)
    |----------------
    | data.aws_subnet_ids.firewall.ids is set of string with 3 elements

Call to function "element" failed: cannot read elements from set of string.
```

```(terraform)
Error: Invalid index

  on test.tf line 43, in output "subnet_ids":
  43:   value = data.aws_subnet.firewall[0].id
    |----------------
    | data.aws_subnet.firewall is object with 3 attributes

The given key does not identify an element in this collection value.
```

```(terraform)
Error: Invalid index

  on test.tf line 42, in output "subnet_ids":
  42:   value = data.aws_subnet_ids.firewall.ids[0]

This value does not have any indices.
```

We can do conversion on the fly to get a particular attribute of an element of the set:

```(terraform)
data "aws_subnet" "firewall" {
  for_each = data.aws_subnet_ids.firewall.ids
  id       = each.value
}

output "subnet_cidr_0" {
  value = [for s in data.aws_subnet.firewall : s.cidr_block][0]
}
```

Which gives us:

```(terraform)
Outputs:

subnet_cidr_0 = 10.11.12.0/27
```

If we simply want one element of the set, we have some options:

```(terraform)
output "subnet_id_0" {
  value = tolist(data.aws_subnet_ids.firewall.ids)[0]
}
```

or

```(terraform)
output "subnet_id_0" {
  value = sort(data.aws_subnet_ids.firewall.ids)[0]
}

```

Both give the same ouput:

```(terraform)
Outputs:

subnet_id_0 = subnet-02d9d3e6004b8cba5
```

However, sorting may change the actual order so maybe we need to more care using sort as a solution.
