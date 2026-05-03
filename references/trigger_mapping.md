# AWS to Azure Trigger Mapping

Every AWS Lambda trigger maps to an Azure HttpTrigger.

| AWS Trigger | Azure Trigger | Route | HTTP Method |
|-------------|---------------|-------|-------------|
| apigateway | HttpTrigger | Preserve original route | Preserve original method |
| sqs | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | defaultHttpMethod |
| sns | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | POST |
| s3 | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | POST |
| eventbridge | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | POST |
| dynamodb | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | POST |
| schedule | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | POST |
| unknown | HttpTrigger | `/<baseRoutePrefix>/<functionName>` | defaultHttpMethod |

## Notes
- ALL triggers map to HttpTrigger in this migration model
- `baseRoutePrefix` and `defaultHttpMethod` come from user inputs
- For apigateway triggers, preserve the original route and method from the SAM/API definition
