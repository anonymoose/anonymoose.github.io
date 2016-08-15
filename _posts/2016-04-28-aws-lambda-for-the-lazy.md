---
layout: post
title: "AWS Lambda for the Lazy"
description: "Using AWS Lambda to create a secure file drop."
date: 2016-08-09 12:00:00 -0400
category: aws
tags: [aws, lambda, python]
---
Sometimes you partner with other organizations to make use of their data. “We’ve got a tarball of a million
flub’s to test your algorithm on, how can we get it to you?” or, “We’ve got a million flubs, do you have an API
we can post to?” Happens all the time, and every time it’s a pain in the butt. The ‘right way’ is to set up individual
IAM accounts for each interaction, connected to individual buckets, maybe set up session tokens and a VPN to your
data source. Then, email around some keys and hope they don’t lose them. Sounds awful. If you’re sophisticated you
can handle all this with some Ansible and hit the button each time your boss comes to you with a new potential partnership.

<br/>
If you’re lazy like me, you make a generic solution that works for everybody with no setup and you don’t ever have to
think about it again. If you’re into it this far, you’re lazy. Read on.

![center](/images/2016-04-28-aws-lambda-for-the-lazy/rube.jpg)


## Lambda to the Rescue

Amazon Lambda is an offering from Amazon where they will run some code in response to an event in the AWS universe. No 
servers, scales forever, only pay for when it runs. Conveniently one of these events is S3 PUT operations. This means 
that we can create a Lambda function that, when an S3 PUT happens on a bucket, it will run and do whatever you want. 
What we want is to grab the file that just arrived and move it somewhere private.


<br/>
[![Lambda!](http://img.youtube.com/vi/AZEn_jrnvlI/0.jpg)](http://www.youtube.com/watch?v=AZEn_jrnvlI "Creepy..")
<br/>

## Make Public and Private Buckets
Log into AWS S3 and create 2 buckets. We’ll name them dropbox-public and dropbox-private. The names don’t matter, 
just pick something sensible that fits in your environment and remember them because you’ll use them a bunch in this exercise.

In the public bucket, set your permissions such that “Everyone” can write to your bucket, but not read.
![center](/images/2016-04-28-aws-lambda-for-the-lazy/one.png)
Next, in the public bucket, tell it to log everything to the private bucket so you can audit things if you have to.
![center](/images/2016-04-28-aws-lambda-for-the-lazy/two.png)
The private dropbox is fine with the defaults. 

<br/>
Double check if you’re paranoid. I’ll wait.


## Set Up IAM Access for your Lambda Function

![center](/images/2016-04-28-aws-lambda-for-the-lazy/three.png)
<br/>

Create an IAM Role named “lambda_dropbox_move”. Give it 2 role policies, each policy grant full access to your lambda 
function to read and write to your buckets. This doesn’t give anyone outside access to anything, it just allows your 
Lambda function to talk to S3, which is the whole point.  

<br/>
Each of these policies are simple JSON snippets. To give Lambda access to the private box, add a Role Policy like so. 
Make sure your resource names match.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::dropbox-private"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::dropbox-private/*"
            ]
        }
    ]
}
```
<br/>
To give Lambda access to the public S3 dropbox, add a Role Policy to “lambda_dropbox_move” like so:
<br/>


```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::dropbox-public"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::dropbox-public/*"
            ]
        }
    ]
}
```

<br/>
Easy.

## Create a Lambda Function to do the Work

Finally some code. Go to Lambda in AWS Console. Create a new Function. Make sure Enable Trigger is checked.
<br/><br/>
![center](/images/2016-04-28-aws-lambda-for-the-lazy/four.png)
<br/><br/>
Name it something you’ll remember. ‘lambda_dropbox_mover’ is a good choice. Pick a language, I like Python.
<br/><br/>
![center](/images/2016-04-28-aws-lambda-for-the-lazy/five.png)
<br/><br/>
Select the role you already created ‘lambda_dropbox_move’.
<br/><br/>
![center](/images/2016-04-28-aws-lambda-for-the-lazy/six.png)
<br/><br/>
Paste in some python to do the work. Make sure the bucket names match your reality.


``` {python}
from __future__ import print_function

import json
import urllib
import boto3

print('Loading function')

s3 = boto3.resource('s3')

def lambda_handler(event, context):
    print("Received event: " + json.dumps(event, indent=2))

    # Get the object name, including path from the private box.
    key = urllib.unquote_plus(event['Records'][0]['s3']['object']['key']).decode('utf8')
    
    try:
        # copy the file from the public box to private, maintaining folder structure.
        s3.Object('dropbox-private', key).copy_from(CopySource='dropbox-public/%s' % key)
        # delete it when copy is complete.
        s3.Object('dropbox-public', key).delete()
        return "OK"
    except Exception as e:
        print(e)
        raise e

```

<br/><br/>

Yes, that was really it. Save it and make sure your trigger is present and pointed to the right place.
<br/><br/>
![center](/images/2016-04-28-aws-lambda-for-the-lazy/seven.png)
<br/><br/>

Money.
<br/><br/>

## Test it Out
S3 put a file using the AWS CLI. Make sure you use your public bucket name.

```bash
aws s3 cp test.txt s3://dropbox-public/some/nested/path/test.txt
```
<br/>

The move should happen almost instantly. Check your private bucket to ensure the file is there.

<br/>
```bash
aws s3 ls --recursive s3://dropbox-private
```
<br/>

Output:

<br/>
```bash
2016-07-02 21:32:57   2.9 MiB some/nested/path/test.txt
Total Objects: 1
   Total Size: 2.9 MiB
```
<br/>

Now see if there is anything in your old bucket. There’s not.
<br/>

~~~ bash
aws s3 ls --recursive s3://dropbox-public
~~~

<br/>

Nada, because your Lambda function deleted it from public.

## What Have We Learned?

Amazon Lambda is pretty slick. Debugging is a challenge and everything happens in the UI, unless you deploy zip files 
full of code via AWS CLI, which is an exercise for the reader. Security is a necessary headache, but once you figure 
it out, rinse and repeat.

## Caveats?
Probably. I might not pass the launch codes to the nukes around like this, 
but in reality, it’s probably good for 95% of the data you’d want to move around the net. 
For the rest, do it ‘the right way’ with VPN’s, key exchanges, and all that.
