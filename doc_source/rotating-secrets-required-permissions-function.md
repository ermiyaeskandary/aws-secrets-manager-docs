# Permissions for rotation<a name="rotating-secrets-required-permissions-function"></a>

Secrets Manager uses a Lambda function to rotate a secret\. The Lambda service assumes an [IAM execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html) and provides those credentials to the code for the Lambda function when it executes\. If you turn on rotation by using the Secrets Manager console, the Lambda function, resource policy, execution role, and execution role inline policies are created for you\. 

If you create the Lambda function another way, you must make sure it has the correct permissions\. You also need to create an execution role and make sure it has the correct permissions\. 

To turn on automatic rotation, you must have permission to create the IAM execution role and attach a permission policy to it\. You need both `iam:CreateRole` and `iam:AttachRolePolicy` permissions\. 

**Warning**  
Granting an identity both `iam:CreateRole` and `iam:AttachRolePolicy` permissions allows the identity to grant themselves any permissions\.

In the resource policy for your Lambda function, we recommend that you include the context key `aws:SourceAccount` to help prevent AWS Lambda from being used as a [confused deputy](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)\. For some AWS services, to avoid the confused deputy scenario, AWS recommends that you use both the [aws:SourceArn](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sourcearn) and [aws:SourceAccount](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sourceaccount) global condition keys\. However, if you include the context key `aws:SourceArn` in your Lambda rotation function policy, the rotation function can only be used to rotate the secret specified by that ARN\. We recommend that you include only the context key `aws:SourceAccount` so that you can use the rotation function for multiple secrets\.

## Lambda function resource policy<a name="rotating_Lambda-resource-policy"></a>

**Example**  
The following policy allows Secrets Manager to invoke the Lambda function specified in the `Resource`\. To attach a resource policy to a Lambda function, see [Using resource\-based policies for AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html)\.  

```
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "secretsmanager.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "LambdaRotationFunctionARN",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "111122223333"
        }
      }
    }
  ]
}
```

Alternately, you can add this permission by running the following AWS CLI command:

```
aws lambda add-permission --function-name LambdaRotationFunctionARN --principal secretsmanager.amazonaws.com --action lambda:InvokeFunction --source-account 111122223333 --statement-id SecretsManagerAccess
```

## Lambda function execution role inline policy<a name="rotating_execution-role-policy"></a>

The following examples show inline policies for Lambda function execution roles\. To create an execution role and attach a permissions policy, see [AWS Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)\.

**Example IAM execution role inline policy for single user rotation strategy**  
For an [Rotate DB credentials](rotate-secrets_turn-on-for-db.md), Secrets Manager creates the IAM execution role and attaches this policy for you\.   
The following example policy allows the function to:  
+ Run Secrets Manager operations for secrets that are configured to use this rotation function\.
+ Create a new password\.
+ Set up the required configuration if your database or service runs in a VPC\. See [Configuring a Lambda function to access resources in a VPC](https://docs.aws.amazon.com/lambda/latest/dg/vpc.html)\.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:DescribeSecret",
                "secretsmanager:GetSecretValue",
                "secretsmanager:PutSecretValue",
                "secretsmanager:UpdateSecretVersionStage"
            ],
            "Resource": "SecretARN"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetRandomPassword"
            ],
            "Resource": "*"
        },
        {
            "Action": [
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DetachNetworkInterface"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

**Example IAM execution role inline policy statement for alternating users strategy**  
For an [Rotate DB credentials](rotate-secrets_turn-on-for-db.md), Secrets Manager creates the IAM execution role and attaches this policy for you\.   
The following example policy allows the function to:  
+ Run Secrets Manager operations for secrets that are configured to use this rotation function\.
+ Retrieve the credentials in the separate secret\. Secrets Manager uses the credentials in the separate secret to update the credentials in the rotated secret\.
+ Create a new password\.
+ Set up the required configuration if your database or service runs in a VPC\. For more information, see [Configuring a Lambda function to access resources in a VPC](https://docs.aws.amazon.com/lambda/latest/dg/vpc.html)\.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:DescribeSecret",
                "secretsmanager:GetSecretValue",
                "secretsmanager:PutSecretValue",
                "secretsmanager:UpdateSecretVersionStage"
            ],
            "Resource": "SecretARN"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "SeparateSecretARN"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetRandomPassword"
            ],
            "Resource": "*"
        },
        {
            "Action": [
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DetachNetworkInterface"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

**Example IAM execution role inline policy statement for customer managed key**  
If you use a KMS key other than the AWS managed key `aws/secretsmanager` to encrypt your secret, then you need to grant the Lambda execution role permission to use the key\.   
The following example shows a statement to add to the execution role policy to allow the function to retrieve the KMS key\.  

```
        {
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:GenerateDataKey"
            ],
            "Resource": "KMSKeyARN"
        }
```