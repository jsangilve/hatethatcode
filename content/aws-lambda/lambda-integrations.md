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

Depending on the requirements, and the longevity of the project, you might be working with one or even both of them. I recently found myself in the situation where it was convenient to support both, so I've been exploring how to write consistent code across lambda functions to make this _infrastructure difference_ — the lambda integration —  less prominent within my lambda code (add disclaimer explaining doesn't apply or cover all the cases).

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


### Parsing and validating using JSON schemas

For consistenly parsing and validating the JSON payloads for different request (HTTP POST, PUT, PATCH), I tend to use JSON schemas. For this purpose, the [AJV](https://github.com/ajv-validator/ajv) library is a good option. The API is simple to use, it has types, and — according to their benchmarks — it's fast (I haven't corroborated this but my guess is that, unless your lambda must be extremely performant, it won't introduce a significant overhead).


Let's start setting up AJV and define a basic JSON schema:

```typescript
describe('test different json parsing options', () => {
	// schema containing 3 string fields, but only two of them are required
  const jsonSchema = {
  	properties: {
  		field1: { type: 'string' },
  		field2: { type: 'string' },
  		field3: { type: 'string' },
  	},
  	required: ['field1', 'field2'],
  };
  const ajv = new Ajv({ allErrors: true });
  const validate = ajv.compile(jsonSchema);

	test('check different json payloads', () => {
		expect(validate({field1: 'foo', field2: 'bar'})).toTruthy()
		expect(validate({field1: 'foo', field2: 'bar', field3: ''})).toBeTruthy()
		expect(validate({field1: 1, field2: 'bar'})).toFalsy()
		expect(validate({field1: 'foo', field3: 'blah'})).toFalsy()
	})
}
```

Then compiled schema is an `Ajv.ValidateFunction` i.e, a predicate to evaluate the json payload. This validated whether a given object matches the specified schema and we have the errors available through the `validate.errors` array (`Ajv.ErrorObject` type). However, this will fail if the provided data is null or string.

Let's then try to write a function that given a nullable json (object or string), check whether validates against a JSON schema.


```typescript
class JSONParserError extends Error {
  constructor(public errors: {errorCode: string, message?: string}[]) {
    super();
  }

}

const imperativeParseJSON = <T = object>(data: object | string | null, validate: Ajv.ValidateFunction): T => {
  if (!data) {
  	throw new JSONParserError({errorCode: 'null_json'});
  }
  
  try {
  	const parsed = typeof data === 'string' ? JSON.parse(data) : data;
  	if (validate(parsed)) {
  		return parsed as T;
  	}
  } catch (e) {
  	throw new JSONParserError([{errorCode: 'malformed_json']);
  }
  
  throw new JSONParserError(validate.errors!.map(({ keyword, message }) => ({ errorCode: keyword, message })));
};
```

In case the code is not self-explanatory, here we defined a function that short-circuits when the payload is null, and will try to parse `data` as a json string before validating against the schema. In case of failure, a `JSONParserError` exception is thrown.

The `JSONParseError` is basically wrapping an array of the type `{ errorCode: string, message?: string}`. I could have extended the JSONParseError and created custom exceptions for `null_json` and `malformed_json`, but I preferred to return only one error for the sake of simplicity. Let's see we can use this function:

```typescript
describe('test different json parsing options', () => {
	// ... setup

  test('parse valid json with imperativeParseJSON', () => {
		const validData = {
			field1: 'foo',
			field2: 'bar',
		}
    const resultObj = imperativeParseJSON(validData, compiledSchema);
    expect(resultObj).toEqual(validData);
    const resultString = imperativeParseJSON(JSON.stringify(validData), compiledSchema);
    expect(resultString).toEqual(validData);
  });

  test('parse invalid json with imperativeParseJSON', () => {
		const invalidData = {
			field2: 1,
			field3: 'bar',
		};
    try {
      const resultObj = imperativeParseJSON(invalidData, compiledSchema);
    } catch (e) {
      const errors = e.errors;
      expect(errors.length > 0).toBeTruthy();
      expect(errors).toContainObject({
        errorCode: 'required',
      });
      expect(errors).toContainObject({
        errorCode: 'type',
      });
    }
  });
}
```

From the caller perspective, a try/catch will be to handle an invalid response:

```typescript
const lambdaHandler: APIGatewayHandler = async event => {
	try {
		const parsed = imperativeParseJSON(event.body)
		// ... do something with the payload
	} catch (e) {
		if (e instanceof JSONParseError) {
			return {
				statusCode: 400,
				body: JSON.stringify({ errors: e.errors })
			}
		}
		// catch any other expected exception that could be thrown within the code
		of the lambda function
	}

}
```

This is just control-flow based on exceptions. This code example is straightforward, but in reality, the complexer a lambda function becomes, the harder is to follow where a specific errors comes from. 

Because I particularly dislike this style, I decided to go for a functional approach using an Either Type (Result Type).

Once again, there are a few options, but [fp-ts]() is probably the more complete functional programming library for Typescript. Let's see how can we redefine the parsing function:


```typescript
import * as E from 'fp-ts/lib/Either';

// created an interface for the error type
interface JSONError {
  errorCode: string;
  message?: string;
}

const semiFunctionalParseJSON = <T = object>(
  data: object | string | null,
  validate: Ajv.ValidateFunction,
): E.Either<JSONError[], T> => {
  if (!data) {
  	E.left([NULL_JSON]);
  }
  
  try {
  	const parsed = typeof data === 'string' ? JSON.parse(data) : data;
  	if (!validate(parsed)) {
  		return E.left(validate.errors!.map(({ keyword, message }) => ({ errorCode: keyword, message })));
  	}
  	return E.right(parsed as T);
  } catch (e) {
  	return E.left([MALFORMED_JSON]);
  }
};
```

This new function doesn't throw exception, but returns an Either type from `fp-ts`. In case of error, we'll return an array of `JSONError`. From the caller perspective, this code can be now used as follows:

```typescript
describe('test different json parsing options', () => {
 // ...

test('parse valid json with semiFunctionalParseJSON', () => {
  const validData = {
  	field1: 'foo',
  	field2: 'bar',
  };
	const resultObject = semiFunctionalParseJSON(validData, compiledSchema);
	expect(whenRight(resultObject)).toEqual(validData);
	const resultString = semiFunctionalParseJSON(JSON.stringify(validData), compiledSchema);
	expect(whenRight(resultString)).toEqual(validData);
})
```
While writing lambda functions with AWS lambda-proxy or lambda (custom) integrations there are few deatail
Consistently parsing and APIEvent (API







