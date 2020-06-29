Title: Uploading files with lambda integration and AWS
Date: 2020-MM-DD
Tags: aws, lambda, typescript, s3
Category: lambda
Authors: José San Gil

TLDR

Consider the average size of the file to upload.
If the file size is within the 10MB limit you can upload the files with one request.
If the file size is over th 10MB limit, you need two requests ( pre-signed url or pre-signed HTTP POST)

Uploading a file is one of those common features (use cases) that almost every web applications needs. From adding a profile picture to importing a CSV is a fairly simple task to do in almost every web framework or library.

I'm working on a side project where uploading and processing files is an essential use case. I decided to go for a serverless approach, and assumed it would be a simple task to upload a file to S3 from the browser. In fact, it isn't complicated, but I actually found a couple of hiccups in the process. There are plenty of blog posts out there on how to upload a file to S3 using and AWS lambdas, but there are a few subtleties that you might want to consider first.

I tried to compile most of my learning in the following blog post:

## First option: Amplify JS

If you're uploading the file from the browser — and particularly if your application requires integration with other AWS service — Amplify is probably a good option.
Some setup using amplify-cli is required, but it's rather simple — unless you decide or need to setup resouces manually.

You will need to install the [amplify-cli](https://docs.amplify.aws/cli/start/install#pre-requisites-for-installation), add the `storage` library and put the file in S3.

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

The documentation for the `put` method looks as follows:

```typescript
/**
 * Put a file in storage bucket specified to configure method
 * @param {string} key - key of the object
 * @param {Object} object - File to be put in bucket
 * @param {Object} [config] - { level : private|protected|public, contentType: MIME Types,
 *  progressCallback: function }
 * @return - promise resolves to object on success
 */
put(key: string, object: any, config?: any): Promise<Object>;
```

At first glance, I found the options to be quite limited, but after having a look at code, I found the following configuration options:

* `contentDispoistion`
* `cache`
* `contentType`
* `contentDisposition`
* `cacheControl`
* `expires`
* `metadata`
* `tagging`
* `acl`

In my case, `metadata` or `tagging` were the required options. Using them is as simple as:

```typescript
const uploadFile = async (filename: string, content: string) => {
  return AWSStorage.put(filename, content, {
    level: 'public',
    // metadata seems to accept an object as long as the values are string
    metadata: {
      language: 'en-US',
      metatagNumber: ,
    },
    // with `tagging` the parameters must be URL encoded
    tagging: new URLSearchParams({ filename }).toString(),
  });
};
```


### Disadvantages: 

- If you do not need any other Amplify's library (API, AI, etc) besides storage, _it might not be the right tool for the job_. You'll be adding circa +100 KB to your bundle to upload a file (overkill). Also, you're forced to add the `Auth` library (Amazon Cognito).
- The setup is simple and convenient, but restrictive. Amplify uploads your files to either `private/COGNITO-USER-ID`, `protected/COGNITO-USER_ID` or `public` path depending on the specified `level` (see the snippet above). This bucket structure certainly makes sense and suffices common use cases, but — as everything in software — might not exactly match your requirements.
- If your app doesn't use AWS Cognito for authentication, some manual setup is required (IAM policies).
- The extra options (metadata, tagging, etc) aren't currently documented (as of June 2020). I doubt they're considered private API but 🤷🏽‍.

### Advantages:

- Easy to setup.
- The library automatically creates a multi-part upload i.e., apps can upload files up to 5 TB — that'd probably be a bad idea, but still possible.


### Second option: upload the file using the API Gateway and a lambda function

In my opinion, this is the most flexible way to upload and process — or trigger the procesing — of files, as long as your application doesn't require to upload files over 10 MB. 

The web application would `POST` the file to a given HTTP endpoint as binary data. 

There are different examples on how to do it out there, but please find below a very simple setup using the [Serverless framework](https://www.serverless.com/):

```yaml
# serverless.yaml
# TODO provide example of serverless configuration
```

This serverless configuration creates a lambda function integrated with the API Gateway using the [lambda proxy integration](). 

The endpoint handles the request, and the payload (the file) gets automatically transformed into a string by the lambda proxy integration (extra configuration is required when using the regular lambda integration). 

We need to configure the API Gateway to accept binary media types i.e., a binary `Content-Type` that should be accepted by the endpoint. 

Please note that `s3:PutObject` and `s3:PutObjectTagging` permissions are required to upload the file and put tags, respectively.

The API Gateway — through the lambda proxy integration — transforms the payload into a `base64` string, when the `Content-Type` header matches the API's binary media types. Otherwise, the proxy integration pass down the payload as string.

For example, let's use `multipart/form-data`. The API's binary media types has been configured to accept `multipart/form-data` (see `serverless.yaml` above).

The lambda function would look like this:

```typescript
const uploadFile: APIGatewayProxyHandler = async (event) => {
  const { file, fields } = await parseFormData(event);
  const tags = { filename: file.filename };
  try {
    await s3Client
      .putObject({
        Bucket: BUCKET_NAME,
        Key: fields.filename || file.filename,
        Body: file.content,
        Tagging: queryString.encode(tags),
      })
      .promise();

    return {
      statusCode: 200,
      body: JSON.stringify({ description: 'file created', result: 'ok' }),
    };

  } catch (_error) {
    // this is not ideal error handling, but good enough for the purpose of this example
    return {
      statusCode: 409, body: JSON.stringify({ description: 'something went wrong', result: 'error' })
    }
  }
};

```

This lambda function parses the APIGatewayProxy event (see the `parseFormData` function in the repository or alternatively check the npm package [lambda-multipart-parser](https://github.com/francismeynard/lambda-multipart-parser)), reads the file and extract other fields from the form data. Then, the file gets uploaded to S3, adding tags, and a customized filename when provided.

You can call the function using the following `curl` request:

```bash
curl -vs --progress-bar -X POST -H "Content-Type: multipart/form-data" -F 'file=@FILE_TO_UPLOAD.mp3' -F 'filename=my_custom_filename.m4a' https://your-api-url-id.execute-api.us-east-1.amazonaws.com/dev/upload
```

### Disadvantages

- API Gateway limits the payload size to [10 MB](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html1). If you try to upload a file that exceeds the 10 MB limit, the API Gateway return HTTP 413: `HTTP content length exceeded 10485760 bytes`.

### Advantages

- Relatively easy to setu.
- 

### Third option: upload the file using a pre-signed URL and the API Gateway

This option employs the AWS's API Gateway and two different lambda functions:

The API Gateway is required for the pre-signed URLs

If you're using a pre-signed URL and the API gatewa, t

This option requires a pair of lambda functions: one to generate pre-signed URLs localhost

### Disadvantages

### Advantages

# Fourth option: upload the file using pre-signed POST and the API Gateway

### References

- CSS Tricks example