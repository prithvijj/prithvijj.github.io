---
title: "Resolving 'provided principal was invalid' for AWS::Lambda::Permission"
date: 2024-03-08T15:02:18-05:00
draft: false
---

## Cross-Posting

- https://dev.to/prithvijj/resolving-provided-principal-was-invalid-for-awslambdapermission-49jp

## Context

My friend puts the following Error Message on a Slack Channel that our Team monitors and provides assistance in


`coolLambdaPermission | UPDATE_FAILED | Resource handler returned message: "The provided principal was invalid. Please check the principal and trying again" (Service: Lambda, Status Code: 400 ...)`


He also provides some details on the Jenkins Job that showed this error message, and wanted to get some help. I was free so I jumped into resolving it.

## Issue

The `AWS::Lambda::Permission` CloudFormation Resource would fail to update within the CloudFormation Stack, due to the Principal provided being invalid (based on the error message), even though the Principal provided within the CloudFormation Template was verified as valid. 

## Debugging

The first step I took was checking the AWS CloudFormation Events Tab to see the error message, and as seen the `AWS::Lambda::Permission` CloudFormation Resource failed to update.

Looking into the CloudFormation Template that initiated the Stack Update, I verified the the `AWS::IAM::Role` used within the `AWS::Lambda::Permission` Principal attribute existed and was valid.

I checked the Policy Permissions and Trust Relationship for the given IAM Role, and compared it with similar CloudFormation Stacks that use the same IAM Role for the `AWS::Lambda::Permission` resource, but things looked fine.

Quick Google-Fu searches later, I didn't really get anywhere. My Manager took a quick look and mentioned that the Lambda Permission has an IAM Role associated, which does not exist since it was deleted, which might be causing issues for CloudFormation Stack Updates :shrug:

:thinking: Since the CloudFormation Template changes involved the following

```diff
# serverless.yml

coolLambdaPermission:
  Type: "AWS::Lambda::Permission"
  Properties:
    FunctionName: ...
    Action: ...
-    Principal: ${self:custom.deletedRole}
+    Principal: ${self:custom.validRole}  

```
I would have expected the new `Valid Role` to **replace** the old `Deleted Role` when the CloudFormation Stack is being updated, which it's exactly trying to do.

For a Sanity check, I went into the Lambda Resource, associated with the Lambda Permission, available under `Lambda -> Configuration Tab -> Permissions Tab -> Resource-based policy statements` and verified the `Deleted Role` is present in the Principal, but the actual IAM Role is deleted.

I concluded that it must be the `AWS::Lambda::Permission` `Physical ID` that might be causing the error and hence suggested my friend to create a PR that updates that resource


```diff
# serverless.yml

- coolLambdaPermission:
+ coolLambdaPermissions:
  Type: "AWS::Lambda::Permission"
  Properties:
    FunctionName: ...
    Action: ...
-    Principal: ${self:custom.deletedRole}
+    Principal: ${self:custom.validRole}  
```

This would mean that CloudFormation would create a new `AWS::Lambda::Permission` Resource with a new `Physical ID`, and the update would go through. Nope :x:, that did not work, It failed with the same error message.

```
The provided principal was invalid. Please check the principal and trying again
```

:question: My next question was, `Can I manually add in the Valid Role to the Lambda Permission?` So I went into the Lambda Console, and attempted to manually add the `Valid Role`, and it failed with the same error. This is actually great.

Next step was to manually remove the `Deleted Role` Principal within the Lambda Console, and try to add the `Valid Role` Principal again, and that worked! :tada:

This seems like the Lambda wants all the Principals provided to be Valid, and only then would it allow for more Principals to be added in.

I suggested my friend to do the following:
- Create a PR that removes the `AWS::Lambda::Permission` altogether

```diff
# serverless.yml

- coolLambdaPermission:
-  Type: "AWS::Lambda::Permission"
-  Properties:
-    FunctionName: ...
-    Action: ...
-    Principal: ${self:custom.deletedRole}
```
- Create a follow up PR that adds the `AWS::Lambda::Permission` Resource back

```diff
# serverless.yml

+ coolLambdaPermission:
+  Type: "AWS::Lambda::Permission"
+  Properties:
+    FunctionName: ...
+    Action: ...
+    Principal: ${self:custom.validRole}
```

:white_check_mark: That worked and the CloudFormation Stack was updated successfully


## Solution

If you have the `AWS::Lambda::Permission` CloudFormation Resource, which contains a Principal that does not exist anymore, and if you want to update the Principal attribute with a `Valid Role`, you'll need to remove the `AWS::Lambda::Permission` CloudFormation Resource first, and then add it back in with the `Valid Role`
i.e.

:x: Replacing the Principal within the CloudFormation Template and deploying it would not work(atleast for me) 

```diff
# serverless.yml

- coolLambdaPermission:
+ coolLambdaPermissions:
  Type: "AWS::Lambda::Permission"
  Properties:
    FunctionName: ...
    Action: ...
-    Principal: ${self:custom.deletedRole}
+    Principal: ${self:custom.validRole}  
```

Rather,

1. :white_check_mark: Removing the CloudFormation Resource and deploying the CloudFormation Template

``` diff
# serverless.yml

- coolLambdaPermission:
-  Type: "AWS::Lambda::Permission"
-  Properties:
-    FunctionName: ...
-    Action: ...
-    Principal: ${self:custom.deletedRole}
```

2. :white_check_mark: Adding back the CloudFormation Resource with the new Principal and then deploying the CloudFormation Template

```diff
# serverless.yml

+ coolLambdaPermission:
+  Type: "AWS::Lambda::Permission"
+  Properties:
+    FunctionName: ...
+    Action: ...
+    Principal: ${self:custom.validRole}
```

## Notes

- I would have assumed that changing the Principal Value within the `AWS::Lambda::Permission` Resource would have replaced it with a `Valid Role`, but the oddities of AWS are a mystery.
- A question you might ponder is why was the `Deleted Role`, deleted? It seems like there are Resources that use the `Deleted Role`? It was because of a migration process that involved removing that `Deleted Role` since it should not be used anymore. Who started the migration process, ummm **me** :grimacing:

## References

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html




<!---
First thing was checked the CloudFormation Stack Event Log to see the error message,

Next was checking the CloudFormation Template that caused the updated, and verifying the AWS::IAM::Role that was used for the principal, whether it exists, turns out it does

Next checked the Policy within the given IAM Role and compared it with other CloudFormation stacks, and it still looks good

My manager took a quick look and suggested it could be because the lambda permission role being used previously does not exist since it was deleted, so maybe that could be the case

For sanity check, went into the Lambda that used the lamdba::permission and verified that the role being used does not exist. 

Alright seems like since the role does not exist, cloudformation might be throwing an error when attempting to replace the Principal but can't due to an invalid principal being valid

I suggested my friend to change the CloudFormation Resource such that it's logical id changes, which should hopefully delete the old lambda permisison resource, and add in the new lambda permission resource , but that deployment also failed with the same error

Next thing was to verify, can I just manually add in the role to verify if the new role works, and after attempting to manually add it in, it fails with the same error.

This gives me the idea to then manually delete the lambda permission role that doesn't exist, and try again, and turns out it worked.

Seems like lambda permission would not allow you to add in more principals if one of the principal is invalid.

So my final suggestion was to remove that specific cloudformation resource and let it deploy, and the add the cloudformation resource back in,
--->

