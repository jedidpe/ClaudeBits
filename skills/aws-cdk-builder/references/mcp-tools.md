# MCP Tools Reference: awslabs.aws-iac-mcp-server

All tools are provided by the `awslabs.aws-iac-mcp-server` MCP server. Configure with:
```json
"awslabs.aws-iac-mcp-server": {
  "command": "uvx",
  "args": ["awslabs.aws-iac-mcp-server@latest"],
  "env": { "FASTMCP_LOG_LEVEL": "ERROR" }
}
```

## Table of Contents
- [Documentation Search Tools](#documentation-search-tools)
- [Local Validation Tools](#local-validation-tools)
- [Troubleshooting Tools](#troubleshooting-tools)

---

## Documentation Search Tools

### search_cdk_documentation
Search the AWS CDK knowledge base (API Reference, Best Practices Guide, Code Samples, CDK-NAG rules).

**Parameters:**
- `query` (required): Search query string

**Search tips:**
- Use construct FQN: `"aws-lambda.Function timeout"`, `"aws-s3.Bucket versioning"`
- Boolean operators: `"DynamoDB AND table AND GSI"`, `"Lambda OR Function environment"`
- Property lookups: `"bucket encryption configuration"`, `"vpc nat gateway"`

**When to use:**
- Before writing any construct code
- When a property name or API is uncertain
- To verify CDK-NAG suppression rule IDs

---

### search_cdk_samples_and_constructs
Search CDK code samples, construct patterns, and community constructs.

**Parameters:**
- `query` (required): Search query
- `language` (optional): `typescript` | `python` | `java` | `csharp` | `go` (default: `typescript`)

**When to use:**
- Finding complete working examples (e.g., "Lambda with DynamoDB event source")
- Discovering L3 constructs that wrap common patterns
- Multi-service architecture examples

---

### search_cloudformation_documentation
Search CloudFormation documentation for resource types, properties, and template syntax.

**Parameters:**
- `query` (required): Search query

**When to use:**
- Looking up CloudFormation resource property names for `CfnResource` overrides
- Understanding underlying resource schemas
- Finding intrinsic function behavior

---

### read_iac_documentation_page
Fetch and read a complete documentation page (CDK or CloudFormation) in full.

**Parameters:**
- `url` (required): URL from search results
- `starting_index` (optional): Character offset for pagination (default: 0)

**When to use:**
- After a search returns a relevant URL — read the full page for complete API details
- Reading CloudFormation resource type references in full
- Accessing CDK-NAG rule documentation

---

## Local Validation Tools

### validate_cloudformation_template
Validate CloudFormation template syntax and schema using `cfn-lint`. Returns errors with line numbers and fix suggestions.

**Parameters:**
- `template_content` (required): CloudFormation template as a JSON or YAML string
- `regions` (optional): List of AWS regions to validate against (e.g., `["us-east-1", "eu-west-1"]`)
- `ignore_checks` (optional): List of cfn-lint check IDs to suppress (e.g., `["W3002"]`)

**Output includes:**
- Error type (ERROR/WARNING/INFO)
- Rule ID (e.g., `E3001`, `W2001`)
- Resource and property path
- Line number
- Suggested fix

**When to use:** After every `cdk synth` — do not proceed to compliance check until this passes clean.

**How to get template content from CDK:**
```bash
cdk synth --no-staging > template.json
# or read from cdk.out/<StackName>.template.json
```

---

### check_cloudformation_template_compliance
Validate template against security/compliance rules using `cfn-guard` (AWS Guard Rules Registry, AWS Control Tower proactive controls).

**Parameters:**
- `template_content` (required): CloudFormation template as JSON/YAML string
- `custom_rules` (optional): Additional cfn-guard rules to apply

**Output includes:**
- Compliance status per resource
- Rule violated
- Severity (CRITICAL/HIGH/MEDIUM/LOW)
- Remediation guidance

**When to use:** After `validate_cloudformation_template` passes — final check before deployment.

**Handling violations:**
- CRITICAL/HIGH: Must fix
- MEDIUM: Fix or document accepted risk
- LOW/INFO: Use judgment; suppress in CDK with justification

---

### cdk_best_practices
Returns comprehensive AWS CDK best practices across 5 categories: application configuration, coding, constructs, security, and testing.

**Parameters:** None

**When to use:** Once per session before implementing. Key guidance includes:
- Use context variables for environment-specific config
- Prefer L2/L3 constructs over L1 (`Cfn*`) constructs
- Use `grant*` methods for IAM rather than inline policies
- Synthesize deterministically (avoid `Date.now()` in resource names)
- Apply CDK Nag for security validation

---

## Troubleshooting Tools

### troubleshoot_cloudformation_deployment
Analyze a failed CloudFormation stack deployment. Pattern-matches against 30+ known failure cases and integrates CloudTrail event analysis.

**Parameters:**
- `stack_name` (required): Name of the failed CloudFormation stack
- `region` (required): AWS region (e.g., `us-east-1`)
- `include_cloudtrail` (optional): Include CloudTrail analysis (default: `true`)

**Requires:** Read-only AWS credentials (`cloudformation:Describe*`, `cloudtrail:LookupEvents`)

**Output includes:**
- Root cause analysis
- Specific error messages from CloudFormation events
- CloudTrail deep links for IAM/API errors
- Step-by-step resolution guidance

---

### get_cloudformation_pre_deploy_validation_instructions
Returns instructions for using CloudFormation's built-in pre-deployment validation (change set creation validation).

**Parameters:** None

**When to use:** When setting up CI/CD pipelines or when local cfn-lint validation isn't sufficient — this enables server-side validation during change set creation.
