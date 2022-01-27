---
layout: post
title: "Terraform for_each & list of maps"
date: 2020-04-18
categories: terraform
github_comments_issueid: 1
---

Writing a module for automating creation of bitbucket repositories I ran into a problem with repository variables. I wanted to use for_each to iterate through a list of maps of the variables. Sommething like this:

```(terraform)
locals {
  json_vars = [
    {
      key     = "AWS_REGION"
      value   = "eu-west-1"
      secured = "false"
    },
    {
      key     = "AWS_REGION_2"
      value   = "eu-west-2"
      secured = "false"
  }]
}
```

Turns out it's not possible - or requires an understand of Terraform that I don't possess. Terraform throws a hissy fit whatever the reason:

```(bash)
The given "for_each" argument value is unsuitable: the "for_each" argument
must be a map, or set of strings, and you have provided a value of type tuple.
```

To make it work the list needs to be a map. Values assigned to the individual maps (0 and 1 in this case), are arbitrary.

```(terraform)
locals {
  json_vars = {
    0 = {
      key     = "AWS_REGION"
      value   = "eu-west-1"
      secured = "false"
    },
    1 = {
      key     = "AWS_REGION_2"
      value   = "eu-west-2"
      secured = "false"
  }}
}
```

Using the map of maps with for_each, looks like this:

```(terraform)
resource "bitbucket_repository_variable" "main" {
  repository = bitbucket_repository.main.id
  for_each   = local.json_vars
  key        = each.value["key"]
  value      = each.value["value"]
  secured    = each.value["secured"]
}
```
