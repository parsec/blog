---
author: "parsec"
date: 2023-02-16
linktitle: Packer Variable Files, and How to Write Them
type:
  - post
  - posts
title: Packer Variable Files, and How to Write Them
subtitle: A Missing Docs Mystery
weight: 10
series:
  - "Missing Docs"
tags:
  - packer
  - gitops
---

---

## Backstory

So, I had to create some AMIs.

"Sure, no big deal", I thought, "I'll use Packer!"

The Packer templates themselves aren't too bad. It kind of sucks to parse through Hashicorp's documentation, but sometimes I just need some really explicit instructions. So I write my templates, they work after some debugging, life is good.

New problem: I need to set passwords for various accounts on these AMIs. Time to use some variables!

Except... [the Hashicorp documentation](https://developer.hashicorp.com/packer/docs/templates/hcl_templates/variables#assigning-values-to-input-variables) on variables is... a little lacking, to be honest. It explains under the **Standard Variable Definitions Files** heading that the format should look like such:

```hcl
image_id = "ami-abc123"
availability_zone_names = [
  "us-east-1a",
  "us-west-1c",
]
```

Easy enough, right? It's just some variables being set in a separate file, then you bring it in with a command like so:

```hcl
$ packer build -var-file="testing.pkrvars.hcl"

Error: Failed preparing provisioner-block "shell" ""

  on jenkins.pkr.hcl line 44:
  (source code not available)

jenkins.pkr.hcl:46,27-40: Unsupported attribute; This object does not have an
attribute named "jenkins_pass"., and 3 other diagnostic(s)
```

So... that error points us to the **Important do not skip** block in the docs:

> Important: Unlike legacy JSON templates the input variables within a variable definitions file must be declared via a variables block within a standard HCL2 template file *.pkr.hcl before it can be assigned a value. Failure to do so will result in an unknown variable error during Packer's runtime.

Okay, sure... I was a little confused here, to be honest. It sounded like I should be changing the name of the variable file to something like `vars.pkr.hcl`, but that didn't work. Then it just asked me to set up `variables` blocks. And putting everything in the variable definition files in a `variables` block didn't work, so what gives?

## The Solution

Now that I read back on it, it makes sense. It did not when I was looking at it. So, dear reader, I hope this will help!

### Defining Your Variables

So now we have some variable definitions; how do we get those values into our Packer templates?

`vars.pkr.hcl` - it can technically be named anything, but this name means I don't get it confused with other stuff.

```hcl
variable "web_pass" {
  type      = string
  default   = ""
  sensitive = true
}

variable "user_pass" {
  type      = string
  default   = ""
  sensitive = true
}
```

Notice that we left the `default` value set to `nil`. Also, since my use case was for passwords, I set the `sensitive` flag. This means when the variable is consumed, it won't be printed in the console or log.

### The Variable Definition File

This file should be named something with the extension `.pkrvars.hcl`. If you want it to be really fancy, as long as you're only using one `pkrvars` file you can do `.auto.pkrvars.hcl`, and Packer will automagically pick up the variables file.

Your file should look something like this:

```hcl
var1        = "hello"
var2        = 12345
var3        = ["a","b","c"]
...
```

Now, this file's purpose is to assign values to existing variables, hence the warning they gave us. So, to make this work for us, we *also* need to define the variables in some kind of `.pkr.hcl` file. For me, this was just a `vars.pkr.hcl` file, like we discussed above.

So, we have two pieces of the puzzle now. Variable declarations, and variable definitions. These definitions will overwrite the default value of our declared variables. But how do we consume them?

### Consuming Variables in Packer Templates

To call a variable in packer, you encapsulate it in `${var.x}`. Not too bad, right?

Example:

```hcl
... ${var.var1}
... ${var.var2}
... ${var.var3}
```

This part is actually way easier than it sounds, especially if you used [Auto-loaded Variable Definitions Files](https://developer.hashicorp.com/packer/docs/templates/hcl_templates/variables#auto-loaded-variable-definitions-files). If you did, when you run `packer build .` it should Just Work.

If not, you'll want to specific your variables file like so:

```shell
$ packer build -var-file=./vars.pkrvars.hcl

[build_output_here]
```

## In Closing

And now, assuming nothing weird happened, you should have a Packer template consuming custom variables at runtime! The beautiful thing about doing vairables this way instead of `locals` is that they can be overwritten at runtime. So, for example:

```hcl
packer build -var="key":"value" .
```

This would overwrite whatever variable name you give it (or create a new one if it doesn't exist, at runtime).

Pretty nifty, right? I think so. But I'll probably be writing a few more of these in the future... These docs could use some help, and if I'm using the tools anyway, I might as well help other people use them :)
