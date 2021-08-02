# Mitigating the risks of logging and debugging your Lambda function<a name="best-practice_lamda-debug-statements"></a>

When you create a custom Lambda rotation function to support your Secrets Manager secret, be careful about including debugging or logging statements in your function\. These statements can cause information in your function to be written to Amazon CloudWatch\. Ensure the logging of information to CloudWatch doesn't include any of the sensitive data stored in the encrypted secret value\. Also, if you do choose to include any such statements in your code during development for testing and debugging purposes, make sure you remove those lines from the code before using the code in production\. Also, remember to remove any logs including sensitive information collected during development after you no longer need it\.

The Lambda functions provided by AWS and [supported databases](intro.md#full-rotation-support) don't include logging and debug statements\. 