# Running Microservices on Lambda
Coming Soon: how to run [Hapi](https://hapijs.com) + [TypeScript](https://www.typescriptlang.org) on Lambda


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



