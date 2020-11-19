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

There are some good blog post out [TODO footnote here] there explaining the differences but they can be summarized as:

- Lambda-proxy integration: the easiest to setup. Almost no configuration required within the API Gateway, and everything happens within the lambda function code.
- Lambda-custom integration: more flexible and customizable from the API Gateway side. Configuration is required for handling HTTP status codes.

Depending on the requirements, and the longevity of the project, you might be working with one or even both. I recently found myself in the situation where it was convenient to support both, so I've been exploring how to write consistent code across my lamda function to make this _infrastructure difference_ — the lambda integration —  less prominent within my lambda code (doesn't apply or cover all the cases).




Briefly introduction to the difference in the request and response.

Show code on how to parse using using JSON schemas and AJV



While writing lambda functions with AWS lambda-proxy or lambda (custom) integrations there are few deatail
Consistently parsing and APIEvent (API







