Title: Functional and semi functional approaches to parse JSON with API Gateway lambda-proxy and lambda (custom) integrations.
Date: 2020-11-23
Tags: aws, lambda, typescript, api-gateway, serverless
Category: lambda
Authors: José San Gil


This blog post explores functional approaches for parsing — and validating — JSON payloads using JSON Schemas while working with AWS API Gateway and lambda.

There are two alternatives for request/response integration: the lambda proxy integration and the lambda (custom) integration. 

You could find some good blog post out there explaining the differences but they can be summarized as:

- Lambda-proxy-integration: the easiest to setup. Almost no configuration required within the API Gateway, and everything happens within the lambda function code.
- Lambda-custom-integration: more flexible and customizable from the API Gateway side. The configuration is required for handling HTTP status codes.

Depending on the requirements, and the longevity of the project, you might be working with one or even both of them. I recently found myself in a situation where it was convenient to support both, so I've been exploring how to write consistent code across lambda functions to make this _infrastructure difference_ — the lambda integration —  less prominent within my lambda code.

This post doesn't cover all the difference between both integrations but presents my take on how to consistently parse JSON payloads for both of them. Let's have a quick look at the main difference in terms of the APIEvent:

For the lambda-proxy integration, the request body is provided as a nullable JSON string, while it's a nullable object for the lambda-custom-integration:

```typescript
// lambda proxy integration
const handlerWithLambdaProxyIntegration: APIGatewayProxyHandler = event => {
  const body: string = event.body;
  // ...
}

// lambda custom integration
const handlerWithLambdaCustomIntegration = event => {
  const body = event.body; // this is an object
  // ...
}
```

You might have noticed that `handlerWithLambdaCustomIntegration` is not properly typed. The types from the `aws-lambda` aim to cover the lambda-proxy-integration, but not the custom one. Hence, I extended the [api-gateway-proxy.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/aws-lambda/trigger/api-gateway-proxy.d.ts) types.

```typescript
type APIEvent = Omit<APIGatewayEvent, 'body'> & {
  body: object | string | null;
};
type APIGatewayResult<T> = Omit<APIGatewayProxyResult, 'body'> & {
  body?: object | string | null;
};
type APIGatewayHandler = Handler<CustomLambda, APIGatewayResult>;
```

Nothing sophisticated here — this only covers the `body` attribute — but a base type that can be used for both integrations.

### Parsing and validating using JSON schemas

For consistently parsing and validating the JSON payloads for different kinds of requests (HTTP POST, PUT, PATCH), I tend to use JSON schemas. For this purpose, the [AJV](https://github.com/ajv-validator/ajv) library is a good option. The API is simple to use, it has types, and — according to their benchmarks — it's fast (I haven't corroborated this, but I guess is that, unless your lambda must be extremely fast, it won't introduce a significant overhead).


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

The compiled schema is an `Ajv.ValidateFunction` i.e, a predicate to evaluate the JSON payload. This validated whether a given object matches the specified schema and we have the errors available through the `validate.errors` array (`Ajv.ErrorObject` type). However, this will fail if the provided data is null or string.

Let's then try to write a function that given a nullable JSON (object or string), parses, and validates against a JSON schema. We'll try to keep as a premise that the caller of this function should get the parsed and validated JSON when the data is valid, or a list of errors when is not.


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

In case the code is not self-explanatory, here we defined a function that short-circuits when the payload is null and will try to parse `data` as a JSON string, before validating against the schema. In case of failure, a `JSONParserError` exception is thrown.

The `JSONParseError` is basically wrapping an array of the type `{ errorCode: string, message?: string}`. I could have extended the JSONParseError and created custom exceptions for `null_json` and `malformed_json`, but I preferred to return only one error for the sake of simplicity. Let's see how can we use this function:

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

From the lambda handler perspective, a `try/catch` is required to handle the exception in case of an invalid JSON:

```typescript
const lambdaHandler: APIGatewayHandler = async event => {
	try {
		const parsed = imperativeParseJSON(event.body)
		// ... do something with the parsed payload
	} catch (e) {
		if (e instanceof JSONParseError) {
			return {
				statusCode: 400,
				body: JSON.stringify({ errors: e.errors })
			}
		}
		// catch any other expected exception that could be thrown within the code of the lambda function
	}

}
```

This is just control-flow based on exceptions. This example is straightforward, but in reality, the more complex a lambda function becomes, the harder is to follow where a specific error came from. Even when this function is certainly not doing [shotgun parsing](http://langsec.org/papers/langsec-cwes-secdev2016.pdf) — the payload won't be processed if it doesn't comply with the schema — the respective error response could get intertwined with the lambda's processing code. 

Because I particularly dislike this style, I decided to try a functional approach using an Either Type (Result Type).

Once again, there are a few options, but [fp-ts](https://gcanti.github.io/fp-ts/modules/Either.ts.html) is probably the most complete functional programming library for Typescript. Let's see how can we redefine the parsing function:


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

This new function doesn't throw exceptions but returns an Either type from `fp-ts`. In case of error (_left_), we'll return an array of `JSONError`, or otherwise the parsed JSON (_right_). From the caller perspective, this code can be now used as follows:

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



// lambda function
import { isLeft } from 'fp-ts/lib/Either';

const lambdaHandler: APIGatewayHandler = async event => {
	const parsed = semiFunctionalParseJSON<MyExpectedType>(event.body)
	if (isLeft(parsed)) {
		return {
			statusCode: 400,
			body: JSON.stringify({ errors: parsed.left })
		}
	}

	// processing payload with parsed.right (MyExpectedType)
  // ...
}
```

In my opinion, this is simpler. Somebody reading this code just need to read the first two lines to see what happens when the payload doesn't match. No try/catch, and no control-flow based on exceptions. 

This example is contrived though, and a very similar result could be achieved without using `fp-ts`. Even with the `imperativeParseJSON` version, we could define our own result type or create a wrapper to catch the exception and return a tuple or a similar type with the errors or the parsed payload.

In my experience, the `isLeft` and `isRight` functions can feel odd or confusing for developers without FP experience, so at this point deciding between `fp-ts` or a custom likewise type boils down to taste and the team's agreement.

Because I feel comfortable with this level of `fp-ts` in my lambda functions, I decided to improve the `semiFunctionalParseJSON` a bit more:


```typescript
const semiFunctionalParseJSONv2 = <T = object>(
  data: object | string | null,
  validate: Ajv.ValidateFunction,
): E.Either<JSONError[], T> => {
  if (!data) {
    return E.left([NULL_JSON]);
  }

  const parseResult = typeof data === 'string' 
    ? E.parseJSON(data, () => [MALFORMED_JSON]) 
    : E.right(data as Json);

  if (E.isLeft(parseResult)) {
    return parseResult;
  }

  return validate(parseResult.right)
    ? E.right((parseResult.right as unknown) as T)
    : E.left(validate.errors!.map(({ keyword, message }) => ({ errorCode: keyword, message })));
};
```

We can get rid of the `try/catch` using the `fp-ts` helper `parseJSON` to return an `Either` type. Under the hood, `parseJSON` is just catching the exception and returning an error accordingly to the `onError` function. At this point, every parsing step inside `semiFunctionalParseJSON2` is using an `Either` type, which we might push a little bit further to transform this function into a pipeline:

```typescript
import * as E from 'fp-ts/lib/Either';
import { pipe } from 'fp-ts/lib/function';


const doParseJSON = (data: Json): E.Either<JSONError[], Json> =>
  typeof data === 'string' ? E.parseJSON(data, () => [MALFORMED_JSON]) : E.right(data as Json);

const validateSchema = <T>(validate: Ajv.ValidateFunction) => (
  json: Json,
): E.Either<JSONError[], T> =>
  validate(json)
    ? E.right((json as unknown) as T)
    : // at this point we know `validate.errors` is not null or undefined
      E.left(validate.errors!.map(fromAjvErrorToJsonError));

const parseJSON = <T = object>(
  data: Json | null,
  schema: JsonSchemaKey | JsonSchemaValidator,
): E.Either<JSONError[], T> => {
  return pipe(
		data, 
		E.fromNullable([NULL_JSON]), 
		E.chain(doParseJSON), 
		E.chain(validateSchema<T>(schema))
	);
};
```


At first, this code looks more difficult. We passed from having a single function (`semiFunctionalParseJSONv2`) to three different functions. However, it's simpler than it looks. We just separated the tasks of our parse function into smaller functions following the [Single-responsability principle](https://en.wikipedia.org/wiki/SOLID) (SRP). The defined pipeline will only succeed if:

1. The provided payload is not nullable.
2. When JSON string, it could be successfully parsed.
3. The parsed object matches the given JSON schema.

Each of these functions returns an `Either` type that gets chained. At runtime, the pipeline won't apply any further operation as soon as one of the steps returns an error. Similarly, because of type safety, each step should comply with the following contract: return a `JSONError` array or `data`.

Is this necessary? do you really need to use a pipe operator? Well, all valid questions with the same answer: it depends. The same results could be achieved without any of the functional abstractions, but in my opinion — and I recognize my own bias because of some relative experience with functional languages — they could bring some clarity to the code. While reading the `parseJSON` code, you can identify the happy path — and the steps involved — while the function signature and each step/function specify the error cases.

If further validation of the JSON is required, new steps could be added to the pipeline seamlessly (as long as the types do not change).

Once again, this example is contrived and other use cases implemented with fp-ts might not result in simpler or more readable code. Similarly. the concepts shown here could be also applied to the lambda function itself. This might be material for another post. For the time being, these are the main takeaways from this post:

### Summary

- While parsing the request payload, the differences introduced by each lambda integration could be abstracted by a properly typed parsing function (avoiding shotgun parsing).
- Validating the payloads (input) with JSON schemas can bring consistency across your lambda code. You can even go further and generate Typescript types from these schemas to make sure, once the input is parsed, your function is working with exactly what's expected (check _Parse, don't validate_ on the _Related articles_ section below).
- Finally, avoid control-flow using exceptions (not only in lambda functions but in general). If you (and your team) are keen on the idea, try `fp-ts` or write a Result type (`Either`) for those operations that can inherently fail as parsing.


### Related articles

- [Lambda-Proxy vs Lambda Integration in AWS API Gateway](https://medium.com/@lakshmanLD/lambda-proxy-vs-lambda-integration-in-aws-api-gateway-3a9397af0e6d)
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- [Think you understand the Single Responsibility Principle?](https://hackernoon.com/you-dont-understand-the-single-responsibility-principle-abfdd005b137)
- [Elegant error handling with the Javascript Either monad](https://jrsinclair.com/articles/2019/elegant-error-handling-with-the-js-either-monad/)
