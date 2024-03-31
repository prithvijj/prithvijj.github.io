---
title: "Importing CloudFormation Resources to help fix deployments to Production"
date: 2024-03-31T15:11:01-04:00
draft: false
---

## Cross-Posting

- https://dev.to/prithvijj/importing-cloudformation-resources-to-help-fix-deployments-to-production-3ejn


## Context

A Frontend Developer in another Team, had merged their PR with the following changes:

```diff
# serverless.yml

+ tags:
+    coolTag: coolValue
+    coolTag2: coolValue2
- resources:
   Description: Cool Description
   Resources:
     CoolS3Bucket:
       Type: AWS::S3::Bucket
       ...
    
     CoolS3BucketPolicy:
       Type: AWS::S3::BucketPolicy
       ...
...
```

It was a simple PR to add in AWS Tags for helping out with visibility of the resources created through AWS CloudFormation and also for cost exploration purposes.

It also removed 1 key called `resources`. `Serverless Framework` enthusiasts will know that `resources` [1] contains the AWS CloudFormation Resources that we'd like to deploy. Since the key got removed, it also removed any resources that were being managed, namely the `CoolS3Bucket` and `CoolS3BucketPolicy`. It contained objects that can only be accessed through AWS CloudFront, using custom headers [2].

After the PR was merged, the new CloudFormation Template was deployed to `Alpha`, `Staging` and then `Production`, and caused the website to go down for a little while :sweat_smile:

While it eventually got fixed by adding in the Bucket Policy manually to the S3 Bucket, we now have a scenario where if we reintroduce the `resources` key (reverting the PR is a quick way), it would cause deployments to Production to fail, because the CloudFormation Template wants to "create" a new resource called `cool-s3-bucket` but fails to "create" it, since it already exists. For fun, I tried to help fix their issue.

## Analysis

The first thing I checked was when the new CloudFormation Template (without any resources) was deployed to update the CloudFormation Stack, did it throw an error when attempting to remove a S3 Bucket that contained objects? Turns out based on the `Events` Tab for the CloudFormation Stack, it did try to delete the S3 bucket, but could not since there were objects within it, which is expected. However the CloudFormation Stack just outright removed the `S3 Bucket` and `S3 Bucket Policy` Resource that it managed. This felt weird because there was an error that occurred with the Deletion of the S3 bucket but the CloudFormation Stack Update silently succeeded stating `Update successful. One or more resources could not be deleted`. I expected an `UPDATE_ROLLBACK_*` to appear, but it didn't :shrug:.

Two options came to mind to help fix the current issue:

1) **Create, Update, Delete, Recreate**

- Create a new S3 Bucket `cool-s3-bucket-2` and S3 Bucket policy entirely
- Update AWS CloudFront to refer to the new S3 Bucket
- Delete the `cool-s3-bucket` Resource
- Recreate the S3 Bucket named `cool-s3-bucket` with the Policy
- Update AWS CloudFront to refer back to the old S3 Bucket
- Delete the new S3 Bucket `cool-s3-bucket-2` so that we're back to the same position we were before the PR was introduced

With this approach, it just works and while there would be a bunch of resource deletions, it should bring things back to normal. I was confident about it, since it looks pretty standard in implementation, while avoiding any downtime.

2) **Import Resources**

I would glance by the `Import CloudFormation Resources` feature, but was suspicious about it, mainly in terms of does it actually work? I had refrained from using it thinking that it would mess up the CloudFormation Resources that were being imported versus the CloudFormation Resources intended to be deployed through `serverless.yml`. But it was an option and so to help alleviate the mystery of Importing CloudFormation Resources, looked at understanding it by creating a simple POC to see if it works.

I setup the following for my POC:
- Manually created the S3 Bucket named `test-cool-s3-bucket`
- Manually setup a random S3 Bucket Policy
- Crafted and deployed a CloudFormation Template with minimal resources (example: Log Group Resource)

I clicked on the `Stack Actions -> Import Resources into stack` option and just read through the simple instructions provided. From there, crafted a template that contained S3 Bucket, and S3 Bucket Policy

```yml
# cloudformation-template.yml converted to json after

Resources:
  TestCoolS3Bucket:
    Type: AWS::S3::Bucket
    Properties: ...

  TestCoolS3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties: ...
```

The Import Wizard asked for the `BucketName` to import and after verifying the list of resources being imported, clicked on the `Import Resources` Button. It was able to successfully import my manually created resources into the CloudFormation Stack. I tried updating values within the CloudFormation Template, and was able to successfully deploy an updated CloudFormation Template.

## Results

:tada: Since the `Import Resources` method POC worked, I suggested a plan of steps that needed to be taken. I Crafted the CloudFormation Templates for all our stages/environments (Example: `Alpha` `Staging` `Production` etc.) and we followed those steps for `Alpha`. Once it had successfully imported the resources, we tried to rebuild the Jenkins pipeline and the job was successful. The `Alpha` CloudFormation Stack was back to normal!

We then asked our OPs team to execute those steps to import the CloudFormation Resources into the CloudFormation Stack for the remaining stages/environments, and verified the Jenkins Deployments were successful all the way till `Production`!

## Things that make you go `hmmmm`

Q) Were there any quality gates as such that would have caught this?

Ans) From what I had observed, seems like the PR being reviewed  was the only quality gate. There weren't any diffs being generated for the current CloudFormation Template with the proposed CloudFormation Template. Action items were created later on, to help create more quality gates for seemingly sensitive infrastructure changes.

## References
1. https://www.serverless.com/framework/docs-providers-aws-guide-resources
2. https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-overview.html#forward-custom-headers-restrict-access