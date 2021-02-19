---
layout: post
title: 'CI/CD with Spinnaker and Terraform '
published: true
---

This post serves as a reflection on microservices infrastructure things I learned during my first job as a backend software engineer at Salesforce’s Infrastructure Secrets. I got to work on cloud infrastructure, some CI/CD, and some data security things like metrics, alerts and monitoring, and HashiCorp Vault. I found that while trying to close stories on sprint boards, I didn’t get the chance to dive deeper into the things I work with, or question design/workflow decisions. Having a week in between my last job and Twitter, I think it’d be nice to reflect a little and look into things I was confused about before but had to brush over for time crunch reasons. Instead of just citing readings to support my opinions (which are neither truly original nor well thought out), I will try to cite some youtube videos I really enjoyed watching as well. This post is definitely not sponsored by InfoQ, I just have been binging it with a passion for some months.

![image](https://user-images.githubusercontent.com/17346982/108554866-b3c22880-72c2-11eb-941e-cc5b555f5db9.png) 
_My view from the Salesforce Bellevue office's coffee bar on a pretty day_

I worked in infrastructure, meaning dealing with how to deploy applications on the cloud, commercial, or first-party -- during my time at Salesforce those were very briefly GCP, AWS, and a bit of 1P.  

The most important steps of the workflow as it appears to me were: provisioning the clusters (for security compliances + common tools), provisioning cloud resources (IAM, KMS, S3 buckets, etc), and deploying containerized applications. 

## Spinnaker
When the order of provisioning infrastructure and containerized applications matters (e.g. a KMS key to encrypt an S3 bucket will need deploying before the S3 bucket, and all AWS resources must be provisioned before an app gets deployed), we can really leverage the use of spinnaker. [Spinnaker](https://spinnaker.io/) is an open-sourced, multi-cloud application management and deployment platform. In simple words, we can use Spinnaker to chain together steps in provisioning infrastructure as well as containerized applications. 

* Spinnaker provides support for various stage types: Render/Deploy manifests, Evaluate expression, manual judgement, and pipeline, which allow calling another pre-defined pipeline

* These pipelines can be terraformed, or templatized for per cluster, per region (any parameters available really), scheduled to run in a CRON job, or by triggers, and there’re helper stage configurations, metrics about a pipeline’s state and function evaluators to pause, halt or rollback deployment. 

![image](https://user-images.githubusercontent.com/17346982/108554957-d0f6f700-72c2-11eb-9397-88e8b8883dda.png)



## Terraform
Not everyone should know how to set up commercial cloud resources so long as they deploy applications there (which is pretty much everyone now). It would be much nicer if there is a common tool within an organization to create uniquely named S3 buckets with the same naming convention, least privilege policies as easy as possible, [19:02 of this talk](](https://youtu.be/j6ow-UemzBc?t=1142)) by Michael Bryzek emphasized this point. [Terraform](https://www.terraform.io/) is great for managing infrastructure as code. 

* After defining all cloud elements in [HCL](https://github.com/hashicorp/hcl) (HashiCorp configuration language) config files, `terraform plan` creates a dependency graph between resources, data and vars to see if anything mismatches, compares previous plan (if exists) vs. current configuration. The output of terraform plan looks something like this

![](https://camo.githubusercontent.com/f969015b8ae18991002bbc79a61ec85bf32817ee3cd7250bcf679099f54583d8/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f7363656e6572792d7075626c69632d6173736574732f7363656e6572795f7265636f7264696e672e737667)
_Somewhat like a git diff (Git and Terraform have a lot of similarities) isn't it? The gist of this `terraform plan` output is to tell you which will be added, changed or destroyed._

* Next step is `terraform apply`, which will actually carry out those CRUD (minus the R) operations planned before. This is actually the stage at which real infrastructure elements are compared against deployed ones, i.e. if you plan to create a new S3 bucket with the same name that an existing one already claimed, `terraform apply` will fail. Otherwise, happy-path wise, you will end up with some output like this. There are definitely dangers in doing this locally, e.g. you and your co-worker start running `terraform apply` at the same time. 

![image](https://user-images.githubusercontent.com/17346982/108554912-c2104480-72c2-11eb-861b-2452832b9355.png)


* To clean up testing resources, it’s best to do a `terraform destroy`. Running these terraform commands in CLI, and then making changes to actual resources to test out the configurations might seem inadequate for reproducibility, and that’s why there’s a better way to do this, with terratest, which allows writing tests in Go to assert expected outputs of terraform plans.

## Nuances
[Terraform states](https://www.terraform.io/docs/language/state/index.html) are rather messy to mux with. During one of my oncalls, we had an incident with changing a module that has hardcoded AWS region, which failed, but then subsequent states assume that the region has to be the hardcoded region. Some juggling in tfstate migration was needed, additional support for terraform destroy for specified resource in Spinnaker pipeline from the terraform team, and more importantly, a way to test infrastructure code beyond “unit testing” with terraform CLI or even terratest by having dev modules for each PR build, decoupling terraform modules -- avoiding “God” module, and having a diverse staging environment (aka, at least more than 1 region of AWS staging env to catch the error earlier could have saved us from taking prod hostage in this case)

There might be some info you might rather not have revealed in terraform state (tfstate) files, especially when those will be displayed on Spinnaker logs, for example, secrets. Thankfully, there is a great tool for this, [tfmask](https://github.com/cloudposse/tfmask) which helps masking sensitive output from `terraform plan` and `terraform apply`.

## What's next?

I promised youtube video citations, but so far we've got only 1, but I guarantee you more for Kubernetes, a bit of Go, and HashiCorp Vault.
