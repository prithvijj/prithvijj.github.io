---
title: "Saving 90% of our AWS Cost using ECR Lifecycle Rules"
date: 2024-03-16T15:04:42-04:00
draft: false
---


## Cross-Posting

- https://dev.to/prithvijj/saving-90-of-our-aws-cost-using-ecr-lifecycle-rules-353f


## Context

For fun, I logged into an AWS account that is used for development purposes. Due to it's purpose, there was a high chance that resources may have been dangling around and not being cleaned up. 

Exploring in the AWS Costs Explorer, I took a quick look at the different services, and I came across the ECR Service, which was around `$16 per day (around $480 per month)`. Switching to `Dimension: Usage Type` seems like most of the cost was for an attribute called `TimedStorage-BytesHrs`. 

I opened random ECR Private Repos that were manually setup before through a bash script. I found one in particular where the ECR Repo had around `9000+ Images`, and most of it was prefixed with tag named as `dev-*`. Just scrolling through each image size was around `600 MB`, and existed as far back as 2021. Oh boy, I had to find out what it costed us.

## Analysis

I had to get the size of the Repository, and hence used the AWS CLI command to retrieve the information using `AWS CloudShell`

```bash
aws ecr describe-images --repository-name 'repo-a' --query 'imageDetails[*].imageSizeInBytes' | jq '.[]' | awk '{sum+=0} END{print sum}'
```

Copied over the Bytes Value, and converting into GB
```
4,236,629,104,429 Bytes/1024 = 4,137,333,109.79 KB
4,137,333,109.79 KB/1024 = 4,040,364.36 MB
4,040,364.36 MB/1024 = 3,945.66 GB
```
Yeah that's a lotta GBs. Taking a quick look at [ECR Pricing](https://aws.amazon.com/ecr/pricing/), it's `$0.10 per GB / month`, so doing some quick maths

```
3,945.66 GB * $0.1 GB per month = $ 394 per month potentially
```

This was just for a single ECR Private Repository, which seemed super weird that we were paying this for a long while. Oh man, if this is for 1 ECR Repository, what about the others?

Quick `ChatGPT` queries later, had got the AWS CLI commands to call

```bash
 #!/bin/bash
 
 repositories=$(aws ecr describe-repositories --query 'repositories[*].repositoryName' --output text)
 
 for repo in $repositories; do
         size=$(aws ecr describe-images --repository-name $repo --query 'imageDetails[*].imageSizeInBytes' | jq 'if length == 0 then 0 else .[] end' | awk '{sum+=$0} END{print sum}')
         leng=$(aws ecr list-images --repository-name $repo | jq '.imageIds | unique_by(.imageDigest) | length')
         echo "Repository: $repo, num-of-images: $leng, Total Size: $size Bytes"
 done
```

Called this script within `AWS CloudShell` and piped the output
```bash
bash cool.sh | tee cool.txt
```

From here I wanted to get a list of ECR Private Repos sorted by `Size` so I used the following
```bash
 awk '{sum=$7/1024/1024/1024} {cost=sum*0.1} {printf("%s\tnum-of-images: %s\t%s GB\t\$%s\n",$2,$4,sum,cost)}' cool.txt | column -t | sort -n -k4 > sorted-cool.txt
```

I was able to get a list of all the ECR Private Repositories, along with the Total size of the given repository, and the potential cost for it

```
# Rough Numbers

...
repo-d,	num-of-images:  3800,  890     GB  $89.0
repo-c,	num-of-images:  3100,  1000    GB  $100.0
repo-b,	num-of-images:  5000,  1600    GB  $160.0
repo-a,	num-of-images:  9800,  3900    GB  $390.0
```

## Where to go from here?

I started with providing visibility, by making a post on the Slack Channel to the Team that owned those ECR Private Repositories and providing them information about what I had found. I also asked for their permission to act on cleaning out those unused resources. This eventually led to a 1 hour meeting where we identified stale images and ECR Private Repos that should definitely be deleted.

Next I gathered screenshots of all the ECR Privates Repos we would intend to purge, to get "evidence" of a before and after comparison.

After all the "evidences" were validated, sent the following message in the Slack Thread.

```
I intend to start this procedure by 12:45PM
```
[:information_source: `Turn the Ship Around!` is a nice book, learned about `I intend to` before doing something.]

I went into the 9 ECR Private Repos and manually added the following Lifecycle Rules

```
1 Untagged    expire | sinceImagePushed(7 days) | untagged
2 dev-*       expire | sinceImagePushed(14 days) | tagged prefix [dev-]
```

This essential means:
- Delete any untagged images that are 7 days old
- Delete any tagged images with prefix `dev-` that are 14 days old


After adding it in, we waited for a couple of days for the Lifecycle Rules to execute.


## Results

We saw some great results

```
# Before:
# Rough Numbers

...
repo-d,	num-of-images:  3800,  890     GB  $89.0
repo-c,	num-of-images:  3100,  1000    GB  $100.0
repo-b,	num-of-images:  5000,  1600    GB  $160.0
repo-a,	num-of-images:  9800,  3900    GB  $390.0
```

```
# After
# Rough Numbers

...
repo-d,	num-of-images:  420,  90     GB  $9.0
repo-c,	num-of-images:  360,  120    GB  $12
repo-b,	num-of-images:  480,  150    GB  $15
repo-a,	num-of-images:  670,  330    GB  $33
```


:money_mouth_face: :tada: Looking at the ECR Costs in Cost Explorer, we saw that the Storage costs of the image in the ECR Private Repos (`Attribute: TimedStorage-ByteHrs`) dropped to around `$1.48 per day ($44.4 per month)` compared to the `$16 per day (around $480 per month)` which was `~90% savings` on our AWS bills! 

It was good to share these results with the Team.
One thing that I'd look forward to how to is how to implement this at scale. [Since that's a lotta savings](https://knowyourmeme.com/memes/thats-a-lotta-damage)




## Things that make you go `hmmmm`

Q) How did this flow under the radar?

Ans) I think that it was a mix of :
- Assumptions that there were checks and balances (like the ECR Lifecycle Rules) were already setup
- Changing the Docker Tags that were being used across the 4 years
- No observability in terms of size of the ECR Repository
- Change in Priorities and Teams might have caused this to flow under the Radar

Q) You mentioned manually adding, Why not automate it using a script?

Ans) I think that it was best to start off with manually adding in the rules to help remove most of the costs, and then acting on how this can be scaled out for all repositories.

Q) You mentioned the ECR Repos manually setup using a bash script? Why not IaC?


Unluckly, when the project initially began, there was a bash script that was provided and run manually to help setup the ECR Private Repos. There is clearly room for improvement in terms of converting it all into IaC that can be managed




## References

- https://aws.amazon.com/ecr/pricing/
- https://knowyourmeme.com/memes/thats-a-lotta-damage