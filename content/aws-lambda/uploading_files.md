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


## Second option: upload the file using the API Gateway and a lambda function

In my opinion, this is the simplest and most flexible way to upload and process — or trigger the procesing — files using AWS... as long as your application doesn't require to upload files over 10 MB. 

The web application would `POST` the file to a given HTTP endpoint as binary data. You can find different example on how to do it out there, but please find below a simple setup using the [Serverless framework](https://www.serverless.com/):

```yaml
# serverless.yaml
# TODO provide example of serverless configuration
```

This serverless configuration creates a lambda function integrated with the API Gateway using the [lambda proxy integration](). It adds a policy attaching the S3 permissions required to upload a file. Please note that `s3:PutObject` and `s3:PutObjectTagging` are required to upload the file and put tags, respectively.

The defined endpoint (`POST /upload`) handles the request and transforms the payload (the file) into a string, through the lambda proxy integration (extra configuration is required when using the regular lambda integration), before passing it to the lambda function. 

We need to configure the API Gateway to accept binary media types i.e., the binary content types that should be accepted by the API.

The API Gateway — through the lambda proxy integration — transforms the payload into a `base64` string, when the `Content-Type` header matches the API's binary media types. Otherwise, the proxy integration pass down the payload as string.

For example, let's use `multipart/form-data`. The API's binary media types has been configured to accept `multipart/form-data` (see `serverless.yaml` above).

The lambda function would look like this:

```typescript
// uploadFile.ts
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

This lambda function parses the APIGatewayProxy event (see the `parseFormData` function in the repository or alternatively check the npm package [lambda-multipart-parser](https://github.com/francismeynard/lambda-multipart-parser)), reads the file and extract other fields from the form data. Then, uploads the file to S3, includes the tags, and a customizes the filename when provided.

You can call the function using the following `curl` request:

```bash
curl -vs --progress-bar -X POST -H "Content-Type: multipart/form-data" -F 'file=@FILE_TO_UPLOAD.mp3' -F 'filename=my_custom_filename.m4a' https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/upload
```

### Disadvantages

- API Gateway limits the payload size to [10 MB](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html1). If you try to upload a file that exceeds the 10 MB limit, the API Gateway return HTTP 413: `HTTP content length exceeded 10485760 bytes`. 

### Advantages

- Relatively easy to setup
- A synchronous flow for uploading and processing files. When the file size doesn't surpass the API Gateway's limit and the required processing can be done quickly, this model is simple to use and integrate with any web application. 
- Authorization can be added using the API Gateway's authorizers.

## Third option: upload the file using a pre-signed URL

This option employs an API Gateway's endpoint integrated with a lambda function. The application client requests a pre-signed URL from the endpoint, passing some parameters as metadata and tags (there are some caveats with this one), and uploads the file sending an `HTTP PUT` request to the pre-signed URL. 

Let's see an example on how to this with the `serverless` framework:

```
# serverless.yaml
Functions:
  # ... other functions
  createUploadURL:
    handler: build.createUploadURL
    events: 
      - http: 
          method: post
          path: createUploadURL
          cors: true
```

The snippet above creates the endpoint in the API Gateway together with a new lambda function called `createUploadURL`.  Before deploying these changes, we'll need define the function:

```typescript
// createUploadUrl.ts
export const createUploadUrl: APIGatewayProxyHandler = async (event) => {
  const { fields } = await parseFormData(event);
  const tags = fields.filename ? { filename: fields.filename }: undefined;
  const expirationInSeconds = 30;
  // presigned url for put operation
  // TODO proper error handling
  const url = await s3Client.getSignedUrlPromise('putObject', {
    // Use the filename provided in the form. Otherwise fallback to the original filename
    Key: fields.filename,
    Bucket: BUCKET_NAME,
    Metadata: tags ? tags : undefined,
    Tagging: tags ? queryString.encode(tags) : undefined,
    Expires: expirationInSeconds,
  });

  return {
    statusCode: 200,
    body: JSON.stringify({
      url,
    }),
  };
};

```

The lambda function fetches the form fields, and uses the `filename` field to create a pre-signed URL. There is caveat though. The `getSignedUrlPromise` ignores some parameters as `Tagging`. This is explicitly stated in the javascript SDK's documentation [documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property). Unfortunately, the typescript definitions for `getSignedUrlPromise` define that function parameters as `any` (`getSignedUrlPromise(operation: string, params: any): Promise<string>`), so you might easily miss this detail. 

- _Note 1_: Even when the documentation lists `Expires` among the parameters that will be ignored, it works. The generated URL (signature), expires after the given number of seconds (30 seconds in the example above).

- _Note 2_: I didn't confirm if this behaviour is consistent across S3 SDKs (python, java, go, etc).

Therefore, if your application doesn't require Tagging — or any other unsupported parameter like SSECustomerKey, ACL, or ContentLength —  using a pre-signed url is a good option. Let's see how it looks:

```bash
curl -vs --progress-bar -X POST -F 'filename=fileWithPresignedURL' https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/createUploadUrl
```

The request above will return a JSON with the following format:

```json
{
  "url": "THE_PRESIGNED_URL"
}
```

Then, we can use another `curl` command to upload a file:

```bash
curl -vs --progress-bar -X PUT --upload-file myFile.m4a $THE_PRESIGNED_URL
```

If your application does require `Tagging`, a pre-signed url might not be the best solution, but there is still a work-around: the client application can provide the tags using the `x-amz-tagging` HTTP header. Before debating about the developer experience of the resulting API, let's see a request example:

```bash
curl -X PUT -H "x-amz-tagging: filename=fileUploadedWithPresignedURL" \
--upload-file myFile.m4a $THE_PRESIGNED_URL
```

In my opinion, this approach is not ideal. The client application not only needs to send two requests to upload the file, but it also must provide the url encoded tags — and/or other parameters — as an HTTP header. Too many implementation leaks.

You might have noticed how the lambda function is also passing the tags to the `Metadata` attribute. Contrary to the `Tagging` parameter, this works out of the box — the uploaded file gets the metadata attributes. No need for encoding.

- _Note_: if you wonder whether to use metadata or tags, there is an [interesting answer](https://stackoverflow.com/questions/42126348/difference-between-object-tags-and-object-metadata) answer in SO on the topic.


### Disadvantages

- The application flow might vary, but uploading a file always requires two requests: one to get the pre-signed url, another to upload the file.
- Tagging doesn't work out of the box. Even when the pre-signed URL is created passing the tags, the uploaded file doesn't get tagged. This [issue](https://github.com/aws/aws-sdk-js/issues/1313) has been reported and, according to the [SDK documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property), it doesn't work because some parameters — like `Tagging` — aren't supported and must be provided as headers.

### Advantages

- The file size can be up to 5 GB.
- The URL expiration feature might be helpful on some specific use cases e.g. a user must be able to upload a file without signing in.
- When uploading big files, the server load — or serveless load 😅 — gets transferred to S3. In a traditional web server approach, your server wouldn't be busy handling the files. Meanwhile, in a serverless approach, your lambda functions wouldn't need to be executed to handle the file upload, which, theoretically, translates into a smaller bill — [S3 doesn't charge for data transfer IN](https://aws.amazon.com/s3/pricing/).
- It can be [combined](https://github.com/prestonlimlianjie/aws-s3-multipart-presigned-upload) with multipart uploads. The maximum file (object) size can be up to 5 TB.

## Fourth option: upload the file using pre-signed POST

This option is very similar to a pre-signed URLs, but allows the client application to upload a file using an HTTP POST request. From the S3 documentation:

 Amazon S3 supports HTTP POST requests so that users can upload content directly to Amazon S3. By using POST, end users can authenticate requests without having to pass data through a secure intermediary node that protects your credentials. Thus, HTTP POST has the potential to reduce latency. 

I decided to try this option because of the issue with `Tagging` and pre-signed URLs. The pre-signed POST supports the parameters expected by the `PutObject` operation. Let's see how it works:

First, let's add a new function to the `serverless.yaml` file:

```yaml
# serverless.yaml
Functions:
  # ... other functions
  createPostUpload:
    handler: build.createPostUpload
    events: 
      - http: 
          method: post
          path: createPostUpload
          cors: true
```

Nothing new here. The function looks quite similar to the pre-signed URL option. The next step is to create the lambda function:

```typescript
```

We use the SDK's function `createPresignedPost`. 

The API requires to define fields and condition that should be authorized as part of the requests. This can goes as specific as preventing the client to define some specific tags



### Disadvatages

- The client application must send two requests: a request to get the pre-signed data (JSON object), and a request to POST the file to S3.
- It might be unfair to call the process complicated, but it
-

### Advantages

- Same advantages that pre-signed URLs
- It's the recommended method to upload a file from a web-form.
- It's probably the most flexible approach. Conditions can be specified on the pre-signed filds to make sure the upload file contains exactly what's expected.


### References

- CSS Tricks example