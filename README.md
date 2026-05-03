# AWS to Azure Dependency Mapper Skill

Maps AWS Lambda triggers, dependencies, and SDK usage to Azure equivalents. Produces a `conversion_plan.json` that downstream skills use for code generation.

## Key Features
- Trigger mapping (apigateway → HttpTrigger, sqs → HttpTrigger, etc.)
- Dependency mapping (aws-sdk → azure-sdk)
- Clean naming rules (strips aws/lambda from all names)
- Class/package naming conventions

## Output
- `./conversion_plan.json`

## File Structure
```
aws-azure-mapper-skill/
├── SKILL.md
├── README.md
└── references/
    ├── conversion_plan_schema.json
    ├── trigger_mapping.md
    ├── dependency_mapping.md
    └── naming_rules.md
```
