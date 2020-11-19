Title: Uploading files with serverless
Date: 2020-07-19
Tags: aws, lambda, typescript, s3
Category: lambda
Authors: José San Gil

Uploading a file is one of those common use cases that almost every web application needs. Regardless of whether you're adding a profile picture, uploading a document, or importing a CSV, it is a fairly simple task in almost every language, framework, or library.

I'm working on a side project where uploading and processing files is a fundamental use case. I went for a serverless approach and assumed that uploading a file to S3 from the browser would be a simple task. In fact, it isn't complicated, but I found a couple of hiccups in the process. There are plenty of blog posts out there on how to upload a file to S3 using AWS lambdas, but there are a few subtleties that you might want to consider first. I tried to compile some of my learnings in the following blog post:

### TLDR 🙈

- Consider the average size of the file to upload.
- If the file size is within the 10MB limit, you can upload the files with one request.
- If the file size is over the 10MB limit, you need two requests ( pre-signed url or pre-signed HTTP POST)


## ① First option: Amplify JS

If you're uploading the file from the browser — and particularly if your application requires integration with other AWS service — Amplify is probably a good option.
Some setup using amplify-cli is required, but it's rather simple — unless you decide or need to setup resouces manually.

You will need to install the [amplify-cli](https://docs.amplify.aws/cli/start/install#pre-requisites-for-installation), add the `storage` library and put the file in S3.

```typescript
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

At first glance, I encountered the options a bit limited, but after having a look at code, I found the following configuration options:

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
    },
    // with `tagging` the parameters must be URL encoded
    tagging: new URLSearchParams({ filename }).toString(),
  });
};
```

### ⚠️ Disadvantages

- If you do not need any other Amplify's library (API, AI, etc) besides storage, _it might not be the right tool for the job_. You'll be adding circa +100 KB to your bundle to upload a file (overkill). Also, you're forced to add the `Auth` library (Amazon Cognito).
- The setup is simple and convenient, but restrictive. Amplify uploads your files to either `private/COGNITO-USER-ID`, `protected/COGNITO-USER_ID` or `public` path depending on the specified `level` (see the snippet above). This bucket structure certainly makes sense and suffices common use cases, but — as everything in software — might not exactly match your requirements.
- If your app doesn't use AWS Cognito for authentication, some manual setup is required (IAM policies).
- The extra options (metadata, tagging, etc) aren't currently documented (as of July 2020). I doubt they're considered private API but 🤷🏽‍.

### Advantages:

- Relatively easy to setup.
- The library automatically creates a multi-part upload i.e., apps can upload files up to 5 TB — which is probably a bad idea, but still possible.


## ② Second option: upload the file using the API Gateway and a lambda function

In my opinion, this is the simplest and most flexible way to upload and process files — or trigger the procesing — using AWS... as long as your application doesn't require to upload files over 10 MB. 

The web application would `POST` the file to a given HTTP endpoint as binary data. You can find different examples on how to do it out there, but please find below a simple setup using the [Serverless framework](https://www.serverless.com/):

```yaml
# serverless.yaml
provider:
  name: aws
  runtime: nodejs12.x
  apiGateway:
    # accepted binary type for file uploads
    binaryMediaTypes:
      - 'multipart/form-data'

functions:
  # option 1: sync upload
  uploadFile:
    handler: build.uploadFile
    events:
      - http:
          method: post
          path: upload
          cors: true

resources:
  Resources:
    MyServerlessExampleBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: serverless-example-bucket

    # define a policy for an existing role
    UploadFilePolicy:
      Type: AWS::IAM::Policy
      Properties:
        PolicyName: UploadObjects
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Sid: LambdaPutObjects
              Effect: Allow
              Action:
                - s3:PutObject
                - s3:PutObjectTagging
              Resource: 
                Fn::Join:
                  - ""
                  - - "arn:aws:s3:::"
                    - Ref: MyServerlessExampleBucket
                    - "/*"
        Roles:
          - serverless-example-dev-us-east-1-role 
```

This serverless configuration creates a lambda function integrated with the API Gateway using the [lambda proxy integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html). It adds a policy attaching the S3 permissions required to upload a file. Please note that `s3:PutObject` and `s3:PutObjectTagging` are required to upload the file and put tags, respectively.

The defined endpoint (`POST /upload`) handles the request and transforms the payload (the file) into a string, through the lambda proxy integration (extra configuration is required when using the regular lambda integration), before passing it to the lambda function. 

We need to configure the API Gateway to accept binary media types i.e., the binary content types that should be accepted by the API.

The API Gateway — through the lambda proxy integration — transforms the payload into a `base64` string when the `Content-Type` header matches the API's binary media types. Otherwise, the proxy integration pass down the payload as string.

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
      statusCode: 409, 
      body: JSON.stringify({ description: 'something went wrong', result: 'error' })
    }
  }
};

```

This lambda function parses the APIGatewayProxy event (see the `parseFormData` function in the [repository](https://github.com/jsangilve/serverless-example) or alternatively check the npm package [lambda-multipart-parser](https://github.com/francismeynard/lambda-multipart-parser)), reads the file, and extract other fields from the form data. Then, uploads the file to S3, includes the tags, and customizes the filename when provided.

You can call the function using the following `curl` request:

```bash
curl -vs --progress-bar -X POST \
  -H "Content-Type: multipart/form-data" \
  -F 'file=@FILE_TO_UPLOAD.mp3' \
  -F 'filename=my_custom_filename.m4a' \
  --url https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/upload
```

### ⚠️ Disadvantages

- API Gateway limits the payload size to [10 MB](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html1). If you try to upload a file that exceeds the 10 MB limit, the API Gateway return HTTP 413: `HTTP content length exceeded 10485760 bytes`. 

### Advantages

- Relatively easy to setup
- A synchronous flow for uploading and processing files. When the file size doesn't surpass the API Gateway's limit and the required processing can be done quickly, this model is simple to use and integrate with any web application. 
- Authorization can be added using the API Gateway's authorizers.

## ③ Third option: upload the file using a pre-signed URL

This option employs an API Gateway's endpoint integrated with a lambda function. The application client requests a pre-signed URL from the endpoint, passing some parameters as metadata and tags (there are some caveats with this one), and uploads the file sending an `HTTP PUT` request to the pre-signed URL. 

Let's see an example on how to this with the `serverless` framework:

```yaml
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
  // expects body to be a JSON containing the required parameters
  const body = JSON.parse(event.body || '');
  const { filename, tags } = getUploadParameters(body);
  const url = await s3Client.getSignedUrlPromise('putObject', {
    Key: filename,
    Bucket: BUCKET_NAME,
    Metadata: tags,
    Tagging: tags ? queryString.encode(body.tags) : undefined,
    Expires: 90,
  });

  return {
    statusCode: 200,
    body: JSON.stringify({
      url,
    }),
  };
};

```

The lambda function fetches the form fields, and uses the `filename` field to create a pre-signed URL. There is caveat though. The `getSignedUrlPromise` ignores some parameters as `Tagging`. This is explicitly stated in the javascript SDK's [documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property). Unfortunately, the typescript definitions for `getSignedUrlPromise` has the function parameters as `any` (`getSignedUrlPromise(operation: string, params: any): Promise<string>`), so you might easily miss this detail. 

- _Note 1_: Even when the documentation lists `Expires` among the parameters that will be ignored, it works. The generated URL (signature), expires after the given number of seconds (90 seconds in the example above).
- _Note 2_: I didn't confirm if this behaviour is consistent across S3 SDKs (python, java, go, etc).

Therefore, if your application doesn't require Tagging — or any other unsupported parameter like SSECustomerKey, ACL, or ContentLength —  using a pre-signed url is a good option. Let's see how it looks:

```bash
curl -vs --progress-bar -X POST \
  -F 'filename=fileWithPresignedURL' \
  --url https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/createUploadUrl
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

- _Note_: if you wonder whether to use metadata or tags, there is an [interesting answer](https://stackoverflow.com/questions/42126348/difference-between-object-tags-and-object-metadata) in SO on the topic.


### ⚠️ Disadvantages

- The application flow might vary, but uploading a file always requires two requests: one to get the pre-signed url, another to upload the file.
- Tagging doesn't work out of the box. Even when the pre-signed URL is created passing the tags, the uploaded file doesn't get tagged. This [issue](https://github.com/aws/aws-sdk-js/issues/1313) has been reported and, according to the [SDK documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getSignedUrl-property), it doesn't work because some parameters — like `Tagging` — aren't supported and must be provided as headers.

### Advantages

- The file size can be up to 5 GB.
- The URL expiration feature might be helpful on some specific use cases e.g. a user must be able to upload a file without signing in.
- When uploading big files, the server load — or serveless load 😅 — gets transferred to S3. In a traditional web server approach, your server wouldn't be busy handling the files. Meanwhile, in a serverless approach, your lambda functions wouldn't need to be executed to handle the file upload, which, theoretically, translates into a smaller bill — [S3 doesn't charge for data transfer IN](https://aws.amazon.com/s3/pricing/).
- It can be [combined](https://github.com/prestonlimlianjie/aws-s3-multipart-presigned-upload) with multipart uploads. The maximum file (object) size can be up to 5 TB.

## ④ Fourth option: upload the file using pre-signed POST

This option is very similar to a pre-signed URLs, but allows the client application to upload a file using an HTTP POST request. From the S3 documentation:

> Amazon S3 supports HTTP POST requests so that users can upload content directly to Amazon S3. By using POST, end users can authenticate requests without having to pass data through a secure intermediary node that protects your credentials. Thus, HTTP POST has the potential to reduce latency. 

I decided to try this option because of the issues with `Tagging` and pre-signed URLs. The pre-signed POST supports the parameters expected by the `PutObject` operation. Let's see how it works:

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

Nothing new here. The function definition is quite similar to the pre-signed URL option. The next step is to create the lambda function: 

```typescript
// createPostUploadURL.ts
export const createPostUploadUrl: APIGatewayProxyHandler = async (event) => {
    // expects body to be a JSON containing the required parameters
    const body = JSON.parse(event.body || '');
    const { filename, tags } = getUploadParameters(body);

    // missing proper error handling
    const postObj = s3Client.createPresignedPost({
      Bucket: BUCKET_NAME,
      Expires: 90, // expiration in seconds
      // matches any value for tagging
      Conditions: tags && [['starts-with', '$tagging', '']],
      Fields: {
        key: filename,
      },
    });

    return {
      statusCode: 200,
      body: JSON.stringify({
        url: postObj.url,
        fields: {
          ...postObj.fields,
          // augment post object with the tagging values
          tagging: tags ? buildXMLTagSet(tags) : undefined,
        },
      }),
    };
};

```

We use the SDK's function `createPresignedPost`. It requires fields and conditions that could be part of the request. The `Conditions` key defines an array of policy conditions that should be met to upload the file. You can find a specific section about conditions in the [AWS S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html#sigv4-ConditionMatching).

In the example above, when tags are provided, the resulting policy expects a `tagging` field with any value (an empty string `''` means any value). However, the S3's `PostObject` operation still requires this `tagging` field to contain the set of tags in the following format:

```xml
<Tagging>
  <TagSet>
    <Tag>
      <Key>Tag Name</Key>
      <Value>Tag Value</Value>
    </Tag>
    ...
  </TagSet>
</Tagging>
```

I added a simple `buildXMLTagSet` function for this purpose:

```typescript
/**
 * Given a set of tags, produces the expected XML tagging set format (as string)
 */
export const buildXMLTagSet = (tagset: Record<string, string>): string => {
  const tags = Object.entries(tagset).reduce(
    (acc, [key, value]) => `${acc}<Tag><Key>${key}</Key><Value>${value}</Value></Tag>`,
    '',
  );

  return `<Tagging><TagSet>${tags}</TagSet></Tagging>`;
};
```

The tagging set could also be generated on the client-side, but given how particular is the format, I prefer — if possible — to keep this logic within the lambda function. However, if the file (object) tags are dynamic and, for instance, should be defined by the user as part of the upload form, I would suggest generating the XML on the client-side.

Finally, the response is a JSON payload containing the URL to post the file, and the file tags — when provided.

```json
{
  "url": "https://s3.amazonaws.com/your-bucket",
  "fields": {
    "key": "file_to_be_uploaded.ext",
    "bucket": "serverless-example-bucket",
    "X-Amz-Algorithm": "AWS4-HMAC-SHA256",
    "X-Amz-Credential": "ASIAYJBFLSRZPBUTNKWB/20200718/us-east-1/s3/aws4_request",
    "X-Amz-Date": "20200718T151836Z",
    "X-Amz-Security-Token": "LONG_SECURITY_TOKEN",
    "Policy": "ENCODED_POLICY",
    "X-Amz-Signature": "SIGNATURE_STRING",
    "tagging": "<Tagging><TagSet><Tag><Key>tagX</Key><Value>valueTagX</Value></Tag></TagSet></Tagging>"
  }
}


```

A web application using this lambda function will fetch the policy, and POST the form containing the file and policy fields to the given URL. Find below a test using the lambda function to upload a file.

```typescript
test('upload file', async () => {
  // request the upload URL and POST policy
  const { data, status } = await axios.post(
    'https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/createPostUploadUrl',
    {
      filename: 'my_rock_song.m4a',
      tags: {
        genre: 'rock',
        year: '1994',
      },
    },
  );

  expect(status).toEqual(200);

  // the form fields contains the policy fields and the file to upload.
  const fields = {
    ...data.fields,
    file: fs.createReadStream('./file_to_upload.m4a'),
  };

  // use the npm package `form-data` to emulate the FormData Web API
  const form = new FormData();
  for (const [key, value] of Object.entries(fields)) {
    form.append(key, value);
  }

  const { status: uploadStatusCode } = await axios.post(data.url, form, {
    // AWS requires the Content-Length header
    headers: {
      ...form.getHeaders(),
      'Content-Length': await getContentLength(form),
    },
  });
  expect(uploadStatusCode).toEqual(204);
});

```

When using the lambda's response in a real web application, the POST policy fields (see `data.fields` above) could be set as hidden fields of the form.


### ⚠️ Disadvantages

- The client application must send two requests: a request to get the pre-signed data (JSON object), and a request to POST the file to S3.
- It might be unfair to call the process complicated, but it's definitely not as simple as some of the previous options.
- When it comes to `tagging` the file, the tags should be provided as an XML. If the file tags aren't defined by the user (dynamic), this logic could live inside the lambda function. Otherwise, the expected tagging set format should be generated in the client side. There already a few abstraction leaks in this option, but, in my opinion, this one is particularly cumbersome.

### Advantages

- Same advantages that pre-signed URLs.
- It's the recommended method to upload a file from a web-form.
- It's probably the most flexible approach. Metadata, and tags could be provided, while conditions to restrict the POST request can be defined. As a result, every object written to the S3 bucket should contain exactly what is expected.


## Conclusion

As usual, there isn't a silver bullet. The average file size should be the first element to take into consideration when choosing between the four options. If you expect to upload files over 10MB, option ② can be discarded. On the other hand, you have option ①. The Amplify framework simplifies things like uploading a file — it eliminates the file size problem using multipart uploads — or implementing authentication and authorization with Cognito. Unfortunately, some things aren't so simple or well documented (tagging), and your application could get tightly coupled to AWS — which might not necessarily be a bad thing. 

Ultimately, you have pre-signed URLs (option ③) and pre-signed POST  (option ④). Both require the client-side application to send two requests: one to get the signed URL, and another to upload the file to AWS S3. From the client-side point of view, it's simpler to deal with pre-signed URLs. The client-application fetches a URL, when required, and uploads the file — an HTTP PUT request with JS — to S3. Unfortunately, some parameters aren't supported (tagging), so workarounds must be employed. The pre-signed POST option is similar but expects a POST request. The payload retrieved from the first request can contain all the information required to upload the file (including tags), it's more flexible, but also more complex.

Once again, no silver bullet, but a bunch of _it depends_.

You can find the code for the lambda functions in the following repository: https://github.com/jsangilve/serverless-example.
 
### References

- [https://blog.webiny.com/upload-files-to-aws-s3-using-pre-signed-post-data-and-a-lambda-function-7a9fb06d56c1](https://blog.webiny.com/upload-files-to-aws-s3-using-pre-signed-post-data-and-a-lambda-function-7a9fb06d56c1)
- [https://blog.bigbinary.com/2018/09/04/uploading-files-directly-to-s3-using-pre-signed-post-request.html](https://blog.bigbinary.com/2018/09/04/uploading-files-directly-to-s3-using-pre-signed-post-request.html)
- [https://www.serverless.com/blog/s3-one-time-signed-url](https://www.serverless.com/blog/s3-one-time-signed-url)
- [https://medium.marqeta.com/pre-signing-aws-s3-urls-136f203a75cd?gi=cf38bc74bdef](https://medium.marqeta.com/pre-signing-aws-s3-urls-136f203a75cd?gi=cf38bc74bdef)
- [https://medium.com/swlh/upload-binary-files-to-s3-using-aws-api-gateway-with-aws-lambda-2b4ba8c70b8e](https://medium.com/swlh/upload-binary-files-to-s3-using-aws-api-gateway-with-aws-lambda-2b4ba8c70b8e)
- [https://medium.com/@aidan.hallett/securing-aws-s3-uploads-using-presigned-urls-aa821c13ae8d](https://medium.com/@aidan.hallett/securing-aws-s3-uploads-using-presigned-urls-aa821c13ae8d)
- [https://sookocheff.com/post/api/uploading-large-payloads-through-api-gateway/](https://sookocheff.com/post/api/uploading-large-payloads-through-api-gateway/)
- [https://css-tricks.com/building-your-first-serverless-service-with-aws-lambda-functions/](https://css-tricks.com/building-your-first-serverless-service-with-aws-lambda-functions/)
