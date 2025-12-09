---
title: "Terraform Hacks - Running In-Line Scripts"
layout: post
date: 2020-05-26
categories: 
  - "automation"
  - "devops"
tags: 
  - "cloud"
  - "devops"
  - "integration"
  - "linux"
  - "scripting"
  - "terraform"
---

[Terraform](https://www.terraform.io/) is great, it's as simple as that, codifying complex infrastructure provisioning in to simple, readable configuration files, however there are some scenarios where you have bespoke requirements that you would like to do in a script that HCL just doesn't offer (a problem that can plague many configuration languages and is slowly trying to be addressed as configuration languages mature more, as a side note check out Brendan Burn's experimental [Configula](https://github.com/brendandburns/configula) project which attempts to address these issues).

As a solution of sorts to this is the somewhat obliquely named **null_resource** which allows us some powerful functionality within Terraform.

NOTE: The sample code used for this post can be found in Github [here](https://github.com/tinfoilcipher/blogexamples/tree/main/terraform-running-inline-scripts).

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/terra.png)

## What Is null_resource?

According to the Terraform documentation:

<blockquote>
  The null_resource resource implements the standard resource lifecycle but takes no further action. The triggers argument allows specifying an arbitrary set of values that, when changed, will cause the resource to be replaced.
  <footer>- https://www.terraform.io/docs/providers/null/resource.html</footer>
</blockquote>

...that particular phrase doesn't really clear things up, at least it didn't for me anyway and the documentation is pretty dense too.

In a nutshell, if you want to say for example, call a script or file(s) on the local system that Terraform is running on, we can use **null\_resource** to do this, using the **local** _Provider_ to interact with the local filesystem.

## null_resource Implementation

To understand what we can do let's look at something abstract rather than try and dip in to specifics, we'll combine our **Provider**, **Variables** and **Main** in to a single file:

```terraform
provider local {}

provider null {}

variable "script_location" {
    type        = string
    description = "script location"
    default     = "watcher.sh"
}

variable "file_watch" {
    type        = string
    description = "script location"
    default     = "watching.conf"
}

resource "null_resource" "run_script" {
    #--Trigger should apply only when script changes
    triggers = {
        script_hash = filemd5(var.file_watch)
    }
    
    #--Run script when the configuration file has changed
    provisioner "local-exec" {
        command = "bash ./${var.script_location}"
    }
}
```

So what's happening here:

- **Line 1**: We're calling the **local** provider, upon **terraform init** this backend is loaded and Terraform gains access to the local filesystem

- **Lines 3-7**: We're defining the location to a shell script, this is where we can define the bulk of our actions, what we actually want to happen (we'll come back to this later)

- **Lines 9-13**: The configuration file we're watching for changes

- **Line 15:** A **null_resource** is defined and named **run_script**

- **Lines 17-19**: We define a **trigger** for this resource to be used, in this case that the MD5 hash for the _file\_watch_ file has changed

- **Lines 21-24**: Should the trigger be met, a _Provisioner_ is then run, this is running a straight command on the shell, executing our script

All of this together in effect means that we can, without using a true Terraform **resource**, we can run any action we like, provided that we can script it. In this scenario we're checking for any change having been made to a configuration file, and if it has been made, we'll execute a script ahead of any of our other resources being provisioned/changed/destroyed by Terraform.

## Conclusion and Considerations

Use of **null_resource** is an advanced functionality and one that you shouldn't really get in to too deeply unless you understand the underpinning functions of Terraform well (or at least to a decent level), it's also one I would avoid using too heavily unless you want to end up in dependency hell, it's also worth keeping in mind that **null_resources** won't render in a very readable format in dependency graphs if you're very invested in using them. If you find that you can't get around a specific scenario without using a **null_resource** it could be just the solution you need, but checking back for new _Resources_ that meet your requirements frequently is recommended so you don't lean too heavily on these hacks.

It's also critical to understand that whilst Terraform will suggest that it is **Replacing** your resource(s) at runtime, keep in mind that this is a _null_ resource and as such there is nothing to actually change, this replacement refers only to a change in the current state file (in this example as the MD5 hash has changed it's representative state will have been changed and been replaced in the state file).
