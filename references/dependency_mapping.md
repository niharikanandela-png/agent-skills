# AWS to Azure Dependency Mapping

| AWS Dependency | Action | Azure Replacement | Maven GroupId | Maven ArtifactId |
|----------------|--------|-------------------|---------------|------------------|
| aws-lambda-java-core | REMOVE | Not needed in Azure Functions | — | — |
| aws-lambda-java-events | REPLACE | Custom DTOs (Request/Response models) | — | — |
| spring-cloud-function-adapter-aws | REPLACE | spring-cloud-function-adapter-azure | org.springframework.cloud | spring-cloud-function-adapter-azure |
| aws-sdk-java S3 (software.amazon.awssdk:s3) | REPLACE | Azure Storage Blob | com.azure | azure-storage-blob |
| aws-sdk-java SQS (software.amazon.awssdk:sqs) | REPLACE | Azure Service Bus | com.azure | azure-messaging-servicebus |
| aws-sdk-java SNS (software.amazon.awssdk:sns) | REPLACE | Azure Service Bus | com.azure | azure-messaging-servicebus |
| aws-sdk-java DynamoDB (software.amazon.awssdk:dynamodb) | REPLACE | Azure Cosmos DB | com.azure | azure-cosmos |
| aws-sdk-java Secrets Manager | REPLACE | Azure Key Vault Secrets | com.azure | azure-security-keyvault-secrets |
| aws-sdk-java SES | REPLACE | Azure Communication Email | com.azure | azure-communication-email |
| boto3 / botocore (Python) | MAP | Map to Java Azure SDK equivalents | — | — |
| @aws-sdk/* (JavaScript) | MAP | Map to Java Azure SDK equivalents | — | — |

## Notes
- If no AWS SDK calls are found in the source, no additional Azure SDK deps are needed
- The base Azure Functions dependencies are always required:
  - `com.microsoft.azure.functions:azure-functions-java-library:3.1.0`
  - `org.springframework.cloud:spring-cloud-function-adapter-azure:4.1.0`
  - `org.springframework.boot:spring-boot-starter` (NOT spring-boot-starter-web)
