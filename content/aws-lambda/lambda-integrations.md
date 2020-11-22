Title: Lambda integrations
Date: 2020-11-19
Tags: aws, lambda, typescript, api-gateway, serverless
Category: lambda
Authors: José San Gil

Candidate titles:

 - Processing API Gateway lambda event with fp-tseeee
 - Functional and semi functional approaches to parse JSON with API Gateway lambda-proxy and lambda (custom) integrations.


Depending on your needs you might need to go for lambda functions using API request from the lambda integration


This blog post explores functional approaches for parsing — and validating — json payloads using JSON Schemas while working with AWS API Gateway and lambda.

There are two alternatives for request/response integration: the lambda proxy integration and the lambda (custom) integration. 

You could find some good blog post out [TODO footnote here] there explaining the differences but they can be summarized as:

- Lambda-proxy-integration: the easiest to setup. Almost no configuration required within the API Gateway, and everything happens within the lambda function code.
- Lambda-custom-integration: more flexible and customizable from the API Gateway side. Configuration is required for handling HTTP status codes.

Depending on the requirements, and the longevity of the project, you might be working with one or even both of them. I recently found myself in the situation where it was convenient to support both, so I've been exploring how to write consistent code across lamda functions to make this _infrastructure difference_ — the lambda integration —  less prominent within my lambda code (add disclaimer explaining doesn't apply or cover all the cases).

(This post doesn't cover all the difference between both integrations, but some of the most obvious ones)

First, let's have a quick introduction to the main difference for the API event in both cases:

- For the lambda-proxy integration, the request body is provided as a json string, while it's it's an object for lambda custom integrations.

```typescript
// lambda proxy integration
const handlerWithLambdaProxyIntegration: APIGatewayProxyHandler = event => {
  const body: string = event.body;
  // ...
}

// lambda custom integration
const handlerWithLambdaCustomIntegration = event => {
  const body: string; event.body; // this is an object
  // ...
}
```

You might have notice that `handlerWithLambdaCustomIntegration` is not properly typed. The types from the `aws-lambda` aim to cover the lambda-proxy-integration, but not the custom one. Hence, I extended the [api-gateway-proxy.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/aws-lambda/trigger/api-gateway-proxy.d.ts) types.

```typescript
type APIEvent = Omit<APIGatewayEvent, 'body'> & {
  body: object | string | null;
};
type APIGatewayResult<T> = Omit<APIGatewayProxyResult, 'body'> & {
  body?: object | string | null;
};
type APIGatewayHandler = Handler<CustomLambda, APIGatewayResult>;
```

—

Nothing sophisticated here — this only covers the `body` attribute — but a base type that can be used for both lambdas with proxy and custom integration.

Before diving into parsing, let's examine another common requirement: authorization. The behaviour for the API Gateway authorizers is also affected by the chosen integration. When using the lambda-proxy-integration, APIEvent an `authorizer` attribute as described by the `aws-lamdda` types:

```typescript

```


There are more differences than those mentioned above, but 

Show code on how to parse using using JSON schemas and AJV

### Parsing and validating using JSON schemas

For consistenly parsing and validating the JSON payloads for different request (HTTP POST, PUT, PATCH), I tend to use JSON schemas. For this purpose, the [AJV]() library is a good option. The API is simple to use, it has types, and — according to their benchmarks — it's fast (I haven't corroborated this, but my guess is that, unless your lambda must be extremely performant, it won't introduce a significant overhead).

Take a



While writing lambda functions with AWS lambda-proxy or lambda (custom) integrations there are few deatail
Consistently parsing and APIEvent (API







