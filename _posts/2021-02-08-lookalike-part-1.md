---
layout: post
title: Find your celebrity lookalike part I
image: /assets/img/2021-02-08-lookalike-part-1/arch_wide.jpg
sitemap: true
---


I am developing an application which helps its users identify their celebrity lookalikes. That means, 
when someone uploads a photo of themselves, the app shows celebrities they look like the most.

## Part I - POC


The purpose of developing this application is for me to practice my AWS skills. When it comes to architecture, I 
might not make the same decisions that I otherwise would when prioritizing user requirements. For example, I decided 
that I want to use API Gateway, SQS, S3 and ECS to solve this problem, but most likely using an ELB would have 
given just as good results.
{:.note title="Disclaimer"}

In this post I would like to share how I reached a proof-of-concept state where I have a very simple web frontend 
and the backend is functional in the sense that it consists of the same AWS services in the same setup as I want to 
use it in the future. For now - to have some basic functionality, the app should be able to tell whether there are 
any presidential candidates of the 2020 US elections on the uploaded image.

## Frontend

Alright, let's get into it and start with the easy part, the frontend! Basically I only wanted to have a form where
the user can select and upload the image and a div to show the progress of the face recognition process. Just that easy!

![2021-02-08-lookalike-part-1/html.png](/assets/img/2021-02-08-lookalike-part-1/html.png){:.center }

## Uploading photos to S3

To back up the Upload button with functionality, we are already on 'AWS lands'. I actually had three choices here:

1. Use the S3 Javascript SDK to talk to S3 directly as shown in a 
[public album example by AWS](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/s3-example-photo-album.html). 
- I decided against this solution as I would have needed to open up my S3 bucket to the public, which is quite a big
 threat. And not just to my wallet! 🙂
2. Use APIGateway to provide a REST endpoint and Lambda to take care of the upload as described 
[in this medium post](https://medium.com/swlh/upload-binary-files-to-s3-using-aws-api-gateway-with-aws-lambda-2b4ba8c70b8e). 
- This is again a suboptimal solution, as 
[Lambda is billed on usage duration](https://aws.amazon.com/lambda/pricing/), basically saying how much memory the 
function has used for how much time measured in GB-seconds. As images can reach several MBs of size, making them flow 
through Lambda is not a good idea.
3. Use API Gateway and Lambda to request an S3 pre-signed URL and give that to the user, so they can upload their photo 
using that particular URL
- Well, you guessed it, this is the most secure and economical solution. 
[Here is an example from AWS documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrlPromise-property) 
on how to use it.

![s3.jpg](/assets/img/2021-02-08-lookalike-part-1/s3.jpg){:.center height="252" width="518"}

Photos can be uploaded to S3 with the help of API Gateway and Lambda
{:.figcaption}

## Message to SQS

Alright. We uploaded the photo to S3, now what? Since we are going to use SQS, there is something to be aware of. 
Queues are asynchronous, meaning that producers and consumers don't know about each other, only the queue it connects 
them. Although this is useful in a lot of cases, here we have to solve a problem.

How will the ECS task be able to respond to the user?
{:.lead}

Fortunately Websockets API Gateway can help us out here. Websockets provide a way for the server to initiate 
communication by assigning every user a unique connection id. So if we're able to pass the connection id to the ECS 
task, we have nothing to worry about.

Let's modify the frontend code to send the request through the Websockets API Gateway endpoint with the uploaded S3 
object key. We can easily process this using a Lambda function and put a message to the SQS queue. Remember, the Lambda 
function will receive the request context (including the connection id) from the API Gateway if the 
"[Lambda proxy integrations](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html)" 
feature is turned on. 

![sqs.jpg](/assets/img/2021-02-08-lookalike-part-1/sqs.jpg){:.center height="252" width="518"}

Message with uploaded image key and connection id put to SQS queue
{:.figcaption}

## Processing the image

As we have a message in the SQS queue with a connection id and the uploaded S3 object key data, it's time to process the
 image.

To identify faces on pictures, I am using a library called 
[face_recognition](https://github.com/ageitgey/face_recognition/). 
The tool has a really simple API allowing to identify faces with just a couple of lines of code.

```python
import face_recognition

def identify_presidents(image_path):
    """
    :param image_path: Path to image
    :return: dict. Keys: 'faces', 'biden', 'trump'. Values: boolean if there are faces/biden/trump on the image.
    """
    ret = {FACES: False, BIDEN: False, TRUMP: False}
    unknown_image = face_recognition.load_image_file(image_path)
    unknown_face_encodings = face_recognition.face_encodings(unknown_image)
    if len(unknown_face_encodings) > 0:
        ret[FACES] = True
        for unknown_face_encoding in unknown_face_encodings:
            known_face_matches = face_recognition.compare_faces(known_faces, unknown_face_encoding)
            if not ret[BIDEN]:
                ret[BIDEN] = bool(known_face_matches[0])
            if not ret[TRUMP]:
                ret[TRUMP] = bool(known_face_matches[1])
    return ret
```

[With another small piece of code](https://github.com/daniel-laszlo/lookalike) to poll the SQS queue, download the 
image, call the above function and respond to the user, we are pretty much done coding here.

When it comes to packaging I am using Docker with a usual flow: 
Dockerfile —build→Docker image—push→Amazon ECR—pull→Amazon ECS.

```bash
# One-time: get credentials to be able to upload images to ECR
aws ecr-public get-login-password --region us-east-1 | docker login -u AWS --password-stdin public.ecr.aws/${REGISTRY_ID}

docker build -t run_model .
docker tag lookalike:latest public.ecr.aws/${REGISTRY_ID}/run_model:latest 
docker push public.ecr.aws/${REGISTRY_ID}/run_model:latest
```

![ecs.jpg](/assets/img/2021-02-08-lookalike-part-1/ecs.jpg){:.center height="252" width="518"}

ECS task downloads & processes image, responds to user
{:.figcaption}

## Some takeaways I learned

**CORS issue**: Although it can happen, that CORS is not set up well in API Gateway, it's a very rare situation and 
usually it's pretty easy to solve it. It's just a click on the console.

![apigcors.png](/assets/img/2021-02-08-lookalike-part-1/apigcors.png){:.center height="250px"}

But more often than not, API Gateway is not the underlying issue. For example, it can happen that a Lambda function is 
failing, thus not responding anything to the frontend, and the browser interpreting as a CORS issue with an error
 message.

No 'Access-Control-Allow-Origin' header is present on the requested resource. If an opaque response serves your needs, 
set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
{:.note title="ERROR"}

In this case, it might be a good idea to look at the CloudWatch logs of the Lambda function.

**Use callback function for NodeJS Lambda functions**: Lambda can be pretty picky about response format and so it makes 
sense to always use the given callback function:

```jsx
exports.handler = function(event, context, callback) {
  const response = {
    statusCode: 200,
    headers: {},
    body: JSON.stringify({"message": "response"})
  };
  callback(null, response);
};
```

## Summary

To close up, here's the architecture diagram (no hand-drawing this time 🙂).

![arch.png](/assets/img/2021-02-08-lookalike-part-1/arch.png){:.lead}

> In **part II**, I am going to bring this project from a Proof-of-concept state to something that's actually usable
> and 
satisfies the initial project target. 🙂 Coming soon!
{:.lead}
