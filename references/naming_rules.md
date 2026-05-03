# Clean Naming Rules for Azure Migration

## CRITICAL RULE
No "aws", "lambda", "Aws", "Lambda" in any Azure-side name — folders, files, classes, packages.

## Examples

| Original AWS Name | Clean Azure Name |
|-------------------|-----------------|
| AwsLambdaHelloHandler | HelloHandler |
| awslambdahello | hello |
| LambdaOrderProcessor | OrderProcessor |
| aws-lambda-hello | hello |
| HelloFunction | hello (folder), HelloHandler (class) |
| AwsLambdaUserService | UserService |
| lambda-payment-processor | payment (folder), PaymentHandler (class) |

## Naming Convention

| Element | Pattern | Example |
|---------|---------|---------|
| Package | `com.migration.<cleanFolderName>` | `com.migration.hello` |
| Folder | lowercase clean name | `hello` |
| Handler class | `<CleanName>Handler` | `HelloHandler` (in `.handler` subpackage) |
| Service class | `<CleanName>Service` | `HelloService` (in `.service` subpackage) |
| Request model | `<CleanName>Request` | `HelloRequest` (in `.model` subpackage) |
| Response model | `<CleanName>Response` | `HelloResponse` (in `.model` subpackage) |

## Algorithm to Clean a Name
1. Remove any prefix: "aws", "Aws", "AWS", "lambda", "Lambda", "LAMBDA"
2. Remove any suffix: "Lambda", "Function" (if class already has Handler/Service/etc.)
3. Remove hyphens and underscores, convert to camelCase/PascalCase
4. Folder name = lowercase version
5. Class name = PascalCase version + appropriate suffix (Handler, Service, etc.)
