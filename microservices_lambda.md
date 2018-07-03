# Running Microservices on Lambda
## Core Techologies
- [Hapi](https://hapijs.com) is my personal favorite NodeJS API framework due to its deeply integrated validation
- [TypeScript](https://www.typescriptlang.org) makes coding in JavaScript *MUCH* easier due to the static type checking it provides.  It really helps your programs scale past the point where you can't keep everything in your head.
- [Lambda](https://aws.amazon.com/lambda) can be used to build resilient, scalable, and economical apps.  Later I will write a post about some improvements I think could be had to make it even more efficient.

## Serverless Hapi
Project Structure/Dependency graph:

(`index.ts` || `index.http.ts`) <- `plugins.ts` <- `routes.ts` (hapi module) <- `handlers/$HANDLER_NAME.ts` -> `services/$SERVICE_NAME.ts`

Any can be dependent on config.  So far this is just at POC status (albeit working well).  Before it can be used in a large project, I need to get dependency injection working, possibly through [Inversify](http://inversify.io).

```TypeScript
import { APIGatewayProxyEvent, APIGatewayProxyResult, Handler } from 'aws-lambda';
import { Server, ServerInjectOptions, ServerInjectResponse } from 'hapi';
import * as qs from 'querystring';

import { plugins } from './plugins';
import { HTTP_BASE_PATH, initConfig } from './services/config';

const server = new Server({
  compression: false,
  routes: {
    cors: true,
  },
});

const initServer = async () => {
  await server.register(plugins, {
    routes: {
      prefix: HTTP_BASE_PATH,
    },
  });
  console.log('hapi plugins initialized');
  await server.initialize();
  console.log('server initialized');
};

// kick off both hapi server initialization and API call to pull config
const initialized = Promise.all([initServer(), initConfig()]);

// AWS Lambda handler
export const handler: Handler<APIGatewayProxyEvent, APIGatewayProxyResult> = async (event, context, callback) => {
  // loading server and config are asynchronous, so make sure they are completed
  await initialized;

  // API Gateway separates the query string from the path to be nice, but hapi wants them together
  const url = event.queryStringParameters ? event.path + '?' + qs.stringify(event.queryStringParameters) : event.path;

  const hapiRequest: ServerInjectOptions = {
    method: event.httpMethod,
    url,
    payload: event.body ? event.body : '',
    headers: event.headers ? event.headers : undefined,
  };

  const hapiResponse = await server.inject(hapiRequest);

  const lambdaResponse: APIGatewayProxyResult = {
    statusCode: hapiResponse.statusCode,
    headers: hapiResponse.headers,
    body: hapiResponse.payload,
  };
  if (lambdaResponse.statusCode !== 200) {
    console.log('ERROR response:', lambdaResponse.body);
  }
  return lambdaResponse;
};
```


## Terraform Infrastructure Deploy
This terraform configuration allows the "infrastructure" side to be deployed once and be relatively stable while letting CodePipeline or any other CI system deploy the code.

```hcl
resource "aws_lambda_function" "api_handler" {
  function_name    = "${var.lambda_name}"
  description      = "${var.lambda_description}"
  handler          = "${var.lambda_handler}"
  runtime          = "${var.lambda_runtime}"
  role             = "${aws_iam_role.lambda_role.arn}"
  memory_size      = "${var.lambda_memory_size}"
  timeout          = "${var.lambda_timeout}"
  filename         = "${path.module}/data/package.zip"
  source_code_hash = "${base64sha256(file("${path.module}/data/package.zip"))}"

  # modify lambda function code through CodePipeline, terraform shouldn't attempt to correct
  lifecycle {
    ignore_changes = [
      "source_code_hash",
      "last_modified",
    ]
  }
}

resource "aws_lambda_alias" "live" {
  name             = "${var.lambda_live_alias}"
  description      = "the active version"
  function_name    = "${aws_lambda_function.api_handler.arn}"
  function_version = "$LATEST"

  # modify lambda alias pointer through CodePipeline, terraform shouldn't attempt to correct
  lifecycle {
    ignore_changes = [
      "function_version",
    ]
  }
}
```

## CodePipeline Code Deploy
Developers can edit a deployment definition file that specifies what version they would like deployed.

```json
{
  "lambdas": [
    {
      "name": "lambda-one-name",
      "version": "1.1.1"
    },
    {
      "name": "lambda-two-name",
      "version": "0.0.1"
    }
  ]
}
```

I wrote a Lambda CodePipeline step that will take this file and ensure the specified lambda functions are the correct version.

1. Read config including lambda function name prefix match defined in the job parameters for security scoping.
2. Pull the CodeCommit artifact from S3 and extract the deploy definition file
3. Fan out and deploy each function concurrently
    1. Validate the lambda function starts with the allowed prefix for security
    2. Get the LIVE alias version and the deployed alias version (desired version alias needs dots replaced with dashes to be valid.)
    3. Depending on if the underlying version of the desired and LIVE aliases matches, do one of the following
        1. Same: No-Op, already deployed to the correct version
        2. Different and desired version exists in the current account: rollback since versions are immutable
        3. Desired version doesn't exist in this AWS account: use a cross account role to pull up the function code from a lower environment
4. Collect the results and put CodePipeline job status



