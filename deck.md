---
theme: gaia
_class: lead
paginate: true
backgroundColor: #fff
marp: true
backgroundImage: url('https://marp.app/assets/hero-background.jpg')
---

![bg left:40% 80%](./tf-logo.png)

# **Tips and Tricks**

for the intrepid nephrologist

---

# What is Terraform?
<!-- That's actually the first tip. What even is Terraform?
We use it to build cloudy things. But understanding the parts is important.

Terraform isn't specific to AWS. It's not even specific to the cloud. It's a tool for writing configuration files and making them true.

The core of TF is the programming language, HCL. If you've written "terraform", you're actually writing HCL. HCL has all your typical language features like scalars, functions, and flow control. But it's also got "blocks". More on that soon.

What TF can do depends on what providers you load, which provide blocks.

We usually just load the AWS provider, so it gets resource blocks for Lambdas and DynamoDBs and all that. But it can configure GitHub, or NU's on-prem load balancers, if you load those providers.

Terraform knows how to read HCL and graph your blocks. This is important: if you need to create a ALB, then a listener on that ALB, then a rule on the listener, there's a certain order you need to make the AWS API calls in. You can't create the listener before the ALB exists -- and TF figures this out for you.
-->
- Hashicorp Configuration Language, or **HCL**
    - Typical language things: variables, scalars, function library
    - But also: **Blocks**
- Providers
    - Define **data source** & **resource** blocks
- Terraform reads HCL and builds a graph of your blocks

So TF is: HCL code using the blocks from a provider to configure stuff, in the right order.

---
<!-- I know most of you are familiar with .tf files organized into a couple folders, so let's not think about that today. Let's look at some HCL instead. -->

```sh
# Resources are config items to create.
resource "aws_kms_key" "key" {
  description = "Key for encrypting my secrets"
}

resource "aws_ssm_parameter" "secure_param" {
  name        = "/my/secret/password"
  description = "A secret password. Ooh!"
  type        = "SecureString"
  value       = "1234"

  # We're setting the key_id argument to the `arn` property from the previious block.
  # A property is like a prop on an object.
  #
  # This is also creating a relationship in the graph: can't create this block 
  # until aws_kms_key.key is created, because we need one of its properties.
  key_id      = aws_kms_key.key.arn
}
```

---
<!-- Let's think about some other types of blocks too. 

Here's a data block. This isn't going to create anything in AWS, but it will lad more
information (as properties) about an existing bucket. This can be helpful if you need
details about shared account resources, or resources created by other projects'
terraform.

There are some data source blocks that don't do lookups. The IAM policies
data source blocks let you write IAM policies in HCL instead of JSON, so your IDE
can tell you when you're screwing it up. These aren't looking anything up in AWS;
they're just providing some guard-rails.
-->
```sh
# Data source block that'll look up an S3 bucket.
# This is useful for resources you _don't_ create, e.g. some other app's bucket.
# This block looks up properties with more information about the S3 bucket.
data "aws_s3_bucket" "my_bucket" {
  bucket = "my-cool-bucket-northwestern"
}

# Data source block that doesn't do any lookups in AWS.
# This is just giving you some guard-rails on an IAM policy JSON doc.
# Your IDE won't know the right formatting for the JSON, but HCL + AWS provider
# can tell it what arguments the block allows, so you'll know quicker when you mess up.
data "aws_iam_policy_document" "bad_idea_policy" {
  statement {
    effect = "Allow"

    actions   = ["s3:*"]
    resources = ["*"]
  }
}
```

---
<!-- _class: lead -->
<!-- There are other types of blocks, but I imagine you're bored now. So on with the show. -->
# Three 🔥 Tips

---
<!-- OK, so the first tip is how to DRY out code. If you need a bunch of a resource
block, they all have a "count" argument. 

You can set the count to an int, or in my case, I'm counting how many items are
in a variable's array. 

When you use count on a block, you get the local count.index variable. Your block is basically in a for() now, and count.index is your i variable.

This changes how you access the block's properties. Since your block is no longer just one block, it becomes an array. You can access _all_ copies' props with *.prop, or you
can access one by array index.

There's a for_each argument too, which works on maps: https://www.terraform.io/docs/language/meta-arguments/for_each.html
-->
# #1 - DRYing out the code
Adhere to the **D**on't **R**epeat **Y**ourself principle.

```sh
variable "runtime_secrets" {
  type        = list(string)
  derfault    = ["SECRET_ONE", "SECRET_TUESDAY", "SECRET_THREE"]
}

# Think: for(i=0; i>count(var.runtime_secrets; i++)
resource "aws_ssm_parameter" "secure_param" {
  count = length(var.runtime_secrets)
  name  = "/my-app/dev/${var.runtime_secrets[count.index]}"
}

# Wrong: aws_ssm_parameter.secure_param.arn
# Right: aws_ssm_parameter.secure_param.*.arn
```

---
<!-- So, CloudFront. It's a big rule breaker. Stuff like IAM is global and exists in no region, but CloudFront is a little more insidious than that: it's stuff *must* exist in us-east-1 for it to work.

You can try to deploy CloudFront in us-east-2. It'll work, right up until it tries to attach to your certificate. "Can't find certificate with a perfectly-valid ARN", it'll say.

Yeah. Because CloudFront and its associated service, Lambda@Edge, are weird. You create everything in us-east-1, and Amazon magically replicates the changes out to every other edge location.

That replication happens on its own time too. Your `terraform apply` command will finish, but CloudFront can take fifteen minutes to finish pushing your changes out to the other locations. There's no status for this or anything -- so if you're seeing "old" config after a "successful" deployment, you've kinda gotta guess if your deployment went sideways or if it's just not replicated yet.

The last -- and worst -- thing is Lambda@Edge CloudWatch logs. These Lambdas aren't normal Lambdas -- these run at edge locations when you attach them to a CloudFront distribution. 

Depending on where your user is located, they may hit `us-west-1`. Or `ap-east-1`. It'll automatically create a CloudWatch log group, with no log expiration timeframe, in that region and start logging. 

Useful if you want the logs. Not so useful when you've got 2GB of CloudWatch logs from 2015 burning a hole in your bill.
-->
# #2 - CloudFront Breaks All the Rules
- "I'm using `us-east-2` so everything can be there" 🚫
  - CloudFront certificates & Lambda@Edge functions must exist in `us-east-1`
- "Terraform says it's deployed, we're good to go!" 🚫
  - CloudFront internally replicates changes from `us-east-1` to other edge locations, **on its own schedule**
- "I can make one log group for my Lambda@Edge function" 🚫🚫🚫
  - Lambda@Edge execution logs go to CloudWatch in the region they ran in. Which can be any region.😩

---
<!-- So here's a Terraform trick for setting up a `us-east-1` provider alongside your default `us-east-2` provider. 

This is how you can create most of your infra in `us-east-2`, but selectively make a certificate in `us-east-1`.
-->
```sh
# Default provider block
provider "aws" {
  region  = var.region
}

# Since it's aliased, you have to explicitly use this on a resource
provider "aws" {
  alias  = "virginia"
  region = "us-east-1"
}

resource "aws_acm_certificate" "certificate_request" {
  # . . .

  # Opt-in to the us-east-1 provider like this!
  providers = {
    aws = aws.virginia
  }
}
```

---
# #3 - Images from Lambda/API GW
- Normally, assets (images, .xslx files) are put in a public S3 bucket and served that way
  - Anyone on the internet can download these, no problem
- But some times, that's bad. 
  - Example: Wildcard photos. Letting anyone on the internet download them will upset people
  - People like the *US Dept of Education's Family Policy Compliance Office*, who enforces FERPA

---
<!-- _class: lead -->
# Problem
Lambdas return a JSON document. 

JSON is text.

So you can't put raw binary data in there; it'll break the JSON parser.

---
# Solution: API Gateway is Clever
```js
// Loaded from a /private/ S3 bucket, which this Lambda has permission to access.
const photoFromS3 = await s3.getObject(params).promise();

// Check that the user is authorized to see the picture...
if (! user.isCool()) {
  throw "ERROR! An uncool user is being uncool!";
}

return {
    headers: { "Content-Type": photoFromS3.ContentType },
    statusCode: 200,

    // This is the magic! 
    // API Gateway now knows to decode the body before serving!
    isBase64Encoded: true,
    body: photoFromS3.Body.toString('base64'),
}
```