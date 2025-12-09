---
title: "Terraform Tricks - Working with Timestamps"
layout: post
date: 2020-10-19
categories: 
  - "automation"
  - "devops"
tags: 
  - "automation"
  - "cloud"
  - "devops"
  - "integration"
  - "terraform"
---

An often required feature of any declarative software or scripting is to work with time values. Much of the time this requirement doesn't crawl out of the woodwork until you've been working with it for a while (at least that's usually my experience). It was a relief to learn that Terraform does have this function, but the use is a little out of the ordinary and takes a bit of getting used to. In this post we'll be looking at how Terraform handles time and date as well as how to manipulate those values.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png)

## Working With _locals_, A Quick Preface

Local values are defined within a configuration as values relative only to that configuration, they differ from Terraform _variables_ and do not require a _Provider_ to be used.

_Locals_ can be defined all at once at any point within a Terraform configuration using a _locals_ stanza, as shown below:

```terraform
locals {
    name    = andy
    website = tinfoilcipher.co.uk
}
```

Once a local value is declared, you can reference it in expressions as local.<value>, E.G. **local.name** or **local.website** from the above example, allowing us to avoid the use of expressions multiple times within the same configuration.

Critically, the use of locals also allows us to fully leverage Terraform's wide library of built-in functions as detailed in the documentation **[here](https://www.terraform.io/docs/configuration/functions.html)**.

## Timestamp - The Basics

Now that we know how to use _locals_, lets widen the use by using the **timestamp** _function_. _Timestamp_ will call the current timestamp in UTC, conforming to [**RFC 3339**](https://tools.ietf.org/html/rfc3339).

For the sake of brevity, lets define a timestamp and pass it directly to an output so we can see that it worked correctly:

```terraform
locals {
    current_time = timestamp()
}

output "current_time" {
    value = local.current_time
}
```

Now if we run an **apply** we can see that the timestamp was correctly captured:

```bash
terraform apply

# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
#
# Outputs:
#
# current_time = 2020-10-19T08:00:00Z
```

## Manipulating Timestamp Times

Simply grabbing the timestamp is all good and well, but it's application is incredibly limited unless you're trying to record the exact time that a resource was provisioned. While we might very well want to do this, it's much more practical to want to the timestamp based on the current time.

Two native functions exist to do this; **timeadd** (which predictably increments the timestamp) and **formatdate** which manipulates the format of the timestamp.

Let's take a look at some examples of using **timeadd** first:

```terraform
locals {
    today              = timestamp()
    tomorrow           = timeadd(local.today, "24h")
    twelve_hours_plus  = timeadd(local.today, "12h")
    ten_minutes_plus   = timeadd(local.today, "10m")
}

output "today" {
    value = local.today
}
output "tomorrow" {
    value = local.tomorrow
}
output "now_plus_twelve_hours" {
    value = local.twelve_hours_plus
}
output "now_plus_ten_minutes" {
    value = local.ten_minutes_plus
}
```

As we can see above, we can increment the time by adding standard time expressions, E.G. 24h, 10m (the full accepted list of units are "ns", "us" (or "µs"), "ms", "s", "m", and "h").

If we apply the above configuration we can see the outputs return the time correctly incremented:

```bash
terraform apply

# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
#
# Outputs:

# today = 2020-10-19T08:00:00Z
# tomorrow = 2020-10-20T08:00:00Z
# now_plus_twelve_hours = 2020-10-19T20:00:00Z
# now_plus_ten_minutes = 2020-10-19T08:10:00Z
```

## Manipulating Timestamp Format

Finally, we'll take a look at manipulating the format of timestamps, despite UTC being the most widely used format in cloud platforms it's not the be all and end all and there are often scenarios where timestamps need to be manipulated in order to be consumed by target systems, for this the **formatdate** function can be used.

In the below example we're going to take the initial UTC timestamp and reformat it a couple of times so we're left only with a few reformatted examples:

```terraform
locals {
    current_timestamp  = timestamp()
    current_day        = formatdate("YYYY-MM-DD", local.current_timestamp)
    current_time       = formatdate("hh:mm:ss", local.current_timestamp)
    current_day_name   = formatdate("EEEE", local.current_timestamp)
}

output "current_timestamp" {
    value = local.current_timestamp
}
output "current_day" {
    value = local.current_day
}
output "current_time" {
    value = local.current_time
}
output "current_day_name" {
    value = local.current_day_name
}
```

If we apply, we can see the outputs:

```bash
terraform apply
#
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
#
# Outputs:
#
# current_timestamp = 2020-10-19T08:00:00Z
# current_day = 2020-10-19
# current_time = 08:00:00
# current_day_name = Monday
```

The **formatdate** function has a myriad of other time specifications which are detailed in the Terraform documentation [here](https://www.terraform.io/docs/configuration/functions/formatdate.html) and can be used with some creativity to generate just about every format you might require.

## Conclusion

As we see, despite being limited, Terraform's timestamp functions with a little creativity can provide most of the functions we might need to generate proper timestamps. In future posts I'll be taking a look at some practical examples of how to put these in to effect.
