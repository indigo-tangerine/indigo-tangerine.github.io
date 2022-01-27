---
layout: post
title: "Terraform Bitbucket Provider - Creating Repository & Deployment Variables"
date: 2020-03-24
categories: terraform bitbucket json jsonencode python
github_comments_issueid: 3
---
The Terraform Bitbucket provider doesn't currently have resources for deployment environments and variables. Manually adding these is painful but there is an API that can help when coupled with Terraform null resource.

So the plan was to use null resource to launch a python script that called the Bitbucket API and pass terraform map/list variables as json. Before tackling the Terraform code I made sure it all worked passing the json variables to python using a bash shell script. 

The trouble started when I tried to pass json from Terraform to Python.

The variables:

```terraform
pipeline_vars = [{
    key     = "AWS_REGION"
    value   = "eu-west-1"
    secured = "false"
  }]
```

In the module:

```terraform
locals {
  pipelines    = jsonencode(var.pipeline_vars)
  build_command   = "python3 ${path.root}/scripts/config.py ${var.workspace} ${bitbucket_repository.main.slug} ${local.oauth_key} ${local.oauth_secret} ${local.pipelines}"
```

Outputs:

```terraform
json = [
  {
    "key" = "AWS_REGION"
    "secured" = "false"
    "value" = "eu-west-1"
  }]
```

So far so good, except the python script keeps failing with various forms of json related errors. I could see the variable getting passed correctly but something was getting lost in translation on the way to Python.

```terraform
(local-exec): Executing: ["/bin/sh" "-c" "python3 ./scripts/config.py my-workspace mytestrepo [{\"key\":\"AWS_REGION\",\"secured\":\"false\",\"value\":\"eu-west-1\"}]"]
module.xops.module.test.null_resource.build[0] (local-exec): usage: config.py [-h]
module.xops.module.test.null_resource.build[0] (local-exec):                  workspace repo_slug client_id client_secret deployments
module.xops.module.test.null_resource.build[0] (local-exec):                  pipeline_vars
module.xops.module.test.null_resource.build[0] (local-exec): config.py: error: argument pipeline_vars: invalid loads value: '[key:AWS_REGION]'
```

By the look of it all the json formatting was getting stripped off. Switch to passing the variable from bash and it works just fine.

Whether this is a bug in Terraform or "as intended" I've no idea. The solution/hack comes from a [Gruntwork post by Yevgeniy Brikman](https://blog.gruntwork.io/yak-shaving-series-2-a-tale-of-12-errors-9e404b718660) (much appreciated, thanks)

Base64 encoding to the rescue! I knew the terraform was creating the json correctly because of the Terraform output and the python errors, so I only needed a way to ensure it didn't lose the formatting in transit. Encoding the variable in Terraform and then decoding it in Python did the trick.

```terraform
locals {
  pipelines_64    = base64encode(jsonencode(var.pipeline_vars))
  build_command   = "python3 ${path.root}/scripts/config.py ${var.workspace} ${bitbucket_repository.main.slug} ${local.oauth_key} ${local.oauth_secret} ${local.pipelines_64}"
}
```

```python
if pipeline_vars_64 is not None:
        # Decode the encode parameters
        pipeline_vars = base64.b64decode(pipeline_vars_64).decode('ascii')
```

```terraform
module.xops.module.test.null_resource.build[0]: Provisioning with 'local-exec'...
module.xops.module.test.null_resource.build[0] (local-exec): Executing: ["/bin/sh" "-c" "python3 ./scripts/config.py my-workspace mytestrepo W3siUHJvZHVjdGlvbiI6e30sIlN0YWdpbmciOnt9LCJUZXN0Ijp7ImRldiI6W3sia2V5IjoidGVzdDEiLCJvcmRlciI6IjEiLCJzZWN1cmVkI]
module.xops.module.test.null_resource.build[0] (local-exec): Get oauth token
module.xops.module.test.null_resource.build[0] (local-exec): Create pipeline variables
module.xops.module.test.null_resource.build[0] (local-exec): Removing pipeline variable: AWS_REGION
module.xops.module.test.null_resource.build[0] (local-exec): Deleted pipeline variable: AWS_REGION
module.xops.module.test.null_resource.build[0] (local-exec): Creating new pipeline variable: AWS_REGION
module.xops.module.test.null_resource.build[0] (local-exec): Created: AWS_REGION
module.xops.module.test.null_resource.build[0]: Creation complete after 3s [id=7405517266235952431]
```

yay!
