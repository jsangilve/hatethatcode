Title: Uploading files with lambda integration and AWS
Date: 2020-MM-DD
Tags: aws, lambda, typescript, s3
Category: lambda
Authors: José San Gil

TLDR

Consider the average size of the file to upload.
If the file size is within the 10MB limit you can upload the files with one request.
If the file size is over th 10MB limit, you need two requests ( pre-signed url or POST signed URL)

Uploading a file is one of those common features (use cases) that almost every web applications needs. From adding a profile picture to importing a CSV is a fairly simple task to do in almost every web framework or library.

I'm working on a side project where uploading and processing files is an essential use case. I'm building the project using a serverless approach, and assumed it would be a simple task to upload a file to S3 from the browser. In fact, it isn't complicated, but I actually found a couple of hiccups in the process. There are plenty of blog posts out there on how to upload a file to S3 using and AWS lambdas, but there are a few subtleties that you might want to consider first.
1-

I tried to compile most of my learning in the following blog post:


First option: Amplify JS

If you're uploading the file from the browser — and specially if your application requires integration with other AWS service — Amplify is probably the easier way to go.
Some setup using amplify-cli is required, but it's rather simple — unless you decide to setup resouces manually.

You'll to install the [amplify-cli](https://docs.amplify.aws/cli/start/install#pre-requisites-for-installation), add the `storage` library and put the file in S3.

```
// App.ts
import AWSStorage from '@aws-amplify/storage';

async function uploadFile(filename: string, file: string ) {
  const result = await AWSStorage.put(filename, file, {
    level: 'private',
    contentType: 'audio/m4a',
  });
  // ... do something with the result
  return result;
}
```


Disadvantages: 

- If you aren't using any other Amplify's library (API, AI, etc) besides storage, it's probably not the right tool for the job. You'll be adding circa X KB to your bundle just to upload a file (overkill). Also, you're forced to add the `Auth` library (Amazon Cognito).
- The setup is simple and convenient, but restrictive. Amplify uploads your files to either `private/COGNITO-USER-ID`, `protected/COGNITO-USER_ID` or `public` path depending on the specified `level` (see the snippet above). This bucket structure certainly makes sense and suffices many use cases, but might not exactly match your requirements.
- If your app doesn't use AWS Cognito for authentication, some manual setup is required (IAM policies).

