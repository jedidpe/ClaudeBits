---
name: aws-cdk-builder
description: "Build well-architected AWS CDK applications using the latest construct APIs, documentation, and best practices. Integrates with the awslabs.aws-iac-mcp-server MCP server to search CDK docs, find construct patterns (e.g. Lambda function with DynamoDB, API Gateway with authorizer), validate CloudFormation templates for syntax errors, check security compliance, and troubleshoot failed deployments. Use when users ask to build or scaffold CDK stacks, write CDK infrastructure code for any AWS service, find CDK construct APIs or usage patterns, validate CloudFormation templates, fix compliance violations in IaC, or troubleshoot CloudFormation deployment failures."
---

# AWS CDK Builder

Uses the `awslabs.aws-iac-mcp-server` MCP server to build correct, secure, well-architected CDK applications. Always use the MCP tools — they provide live documentation, real construct APIs, and local validation rather than relying on potentially stale model knowledge.

## Workflow

Follow these steps for every CDK build task:

### Step 1: Clarify and scope

Identify the language (default: TypeScript), target services, and any constraints (VPC, compliance requirements, existing stacks). Ask only if genuinely ambiguous.

### Step 2: Get best practices

Call `cdk_best_practices` at the start of every session. This returns the authoritative CDK guidelines for security, constructs, testing, and application configuration. Apply these throughout.

### Step 3: Find constructs and patterns

Before writing code, search for existing constructs:

```
search_cdk_samples_and_constructs(query="<pattern>", language="typescript")
search_cdk_documentation(query="<ConstructName> <property>")
```

Use boolean operators for precision: `"Lambda AND DynamoDB"`, `"S3 AND encryption AND bucket"`. If a search result URL looks promising, call `read_iac_documentation_page(url=<url>)` to read the full page.

### Step 4: Implement

Write CDK code using the exact APIs from Step 3. Follow the patterns in `references/patterns.md` for common architectures.

### Step 5: Validate the synthesized template

After writing CDK, synthesize to CloudFormation and validate:

```
validate_cloudformation_template(template_content="<synth output>")
```

Fix every error before proceeding. The tool returns line numbers and specific fix suggestions.

### Step 6: Check compliance

Run security/compliance checks on the final template:

```
check_cloudformation_template_compliance(template_content="<synth output>")
```

Address all HIGH and CRITICAL violations. Document any accepted MEDIUM/LOW risks.

### Step 7: Troubleshoot (if deployment fails)

If a deployed stack fails:

```
troubleshoot_cloudformation_deployment(stack_name="<name>", region="<region>")
```

The tool pattern-matches against 30+ known failure cases and includes CloudTrail analysis.

## MCP Tool Reference

See `references/mcp-tools.md` for complete parameter details and usage tips for all 9 tools.

## Common Patterns

See `references/patterns.md` for CDK code patterns covering:
- Lambda + DynamoDB
- API Gateway + Lambda + Authorizer
- ECS Fargate + ALB
- S3 + CloudFront
- VPC + Security Groups
- Step Functions workflows

## Key Principles

- **Never guess construct APIs** — always search documentation first
- **Always validate** — run both `validate_cloudformation_template` and `check_cloudformation_template_compliance`
- **L2/L3 constructs first** — prefer high-level constructs over `CfnResource` unless unavoidable
- **Least privilege IAM** — use `grant*` methods (e.g., `table.grantReadWrite(fn)`) instead of manual policy statements
- **Removal policies** — set explicit `RemovalPolicy` for stateful resources (databases, S3 buckets)
- **CDK Nag** — for production stacks, suppress rules explicitly with justification comments
