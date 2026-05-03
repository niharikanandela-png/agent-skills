---
name: aws-azure-dependency-mapper
description: >-
  Maps AWS Lambda dependencies, triggers, and SDK usage to Azure equivalents.
  Strips AWS/Lambda from names. Produces conversion_plan.json.
  USE FOR: dependency mapping, trigger mapping, AWS to Azure migration planning, conversion plan.
---

# AWS to Azure Dependency Mapper Skill

## Context

You are an expert cloud migration architect. Your job is to analyze discovered AWS Lambda functions and produce a complete Azure conversion plan. You do NOT generate any code.

Use relative paths only.

## Inputs

| Input | Description |
|-------|-------------|
| lambda_inventory.json | File produced by the Lambda Scanner skill |
| javaVersion | Target Java version (default: 17) |
| framework | Target framework (default: spring-cloud-function) |
| executionModel | Execution model (default: http-per-function) |
| preserveFunctionNames | Whether to preserve original names (boolean) |
| baseRoutePrefix | Base route prefix for HTTP endpoints (e.g., `/api`) |
| defaultHttpMethod | Default HTTP method when trigger is unknown (e.g., `POST`) |
| azureFunctionAppName | Target Azure Function App name |

## Procedure

### Step 1: Read Inventory
Read `./lambda_inventory.json` from the current directory.

### Step 2: Apply Trigger Mapping
Use the mapping table in `references/trigger_mapping.md` for each function.

### Step 3: Apply Dependency Mapping
Use the mapping table in `references/dependency_mapping.md` for each function.

### Step 4: Apply Clean Naming Rules
Use the rules in `references/naming_rules.md` — CRITICAL: no "aws", "lambda" in any Azure name.

### Step 5: Read Original Source Files
- Read source files listed in `sourceFiles` — ONE at a time, max 3
- Identify AWS SDK method calls, request/response shapes, business logic

## Output

Write to: `./conversion_plan.json` — see `references/conversion_plan_schema.json` for format.

## Constraints

- Do NOT generate any Java code
- Do NOT generate pom.xml or project files
- Do NOT skip any function from the inventory
- Do NOT include "aws" or "lambda" in any Azure-side name
- If a mapping cannot be determined, use `"manual_review_required"`
- Every Lambda MUST map to an HttpTrigger
