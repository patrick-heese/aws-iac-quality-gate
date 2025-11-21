# Reusable IaC Quality Gate (GitHub Actions)  
This repository provides a single reusable GitHub Actions workflow that standardizes **IaC quality checks** across repos. It runs linting and policy scans, produces plans/change sets as artifacts, and can optionally apply behind GitHub Environment approvals using OIDC-assumed AWS roles.  
- **CloudFormation**: **cfn-lint**, optional **Checkov**, **Change Set** creation (no-execute), artifact upload, optional **execute** (env-gated).  
- **SAM**: `sam validate --lint`, optional **Checkov**, `sam build`, **Change Set** creation (no-execute), artifact upload, optional **deploy** (env-gated).  
- **Terraform**: `fmt`, `validate`, **TFLint**, optional **Checkov**, **plan (JSON)**, optional **Conftest** on the plan, artifact upload, optional **apply** (guarded by environment approvals).  

## Project Structure  
```plaintext
aws-iac-quality-gate
├── .github/workflows/					  # GitHub Actions workflows
│   └── reusable-iac-gate.yml     # IaC Quality Gate
├── LICENSE
└── README.md
```

## How it Works  
- Caller repos **invoke** this workflow using `workflow_call`.  
- The gate runs tool-specific **linting + policy checks**, produces **plans/changesets** as artifacts, and (optionally) **applies** behind a protected GitHub **Environment** (e.g., `prod`) via **OIDC-assumed** AWS roles.  
- **Conftest** is supported for:  
    - **CloudFormation/SAM**: directly against the template file  
    - **Terraform**: against `tfplan.json`  
- Setting `environment: prod` binds the job to the **GitHub Environment** named `prod`:  
    - Environment-scoped **secrets** (if any) are exposed to the job.  
    - **Protection rules** (e.g., required reviewers / wait timers) **gate** the job.  
    - Approvals matter primarily when `apply: true`.  
- If you don't need gating, omit `environment` or create an unprotected environment.  

## Inputs  
- `iac_tool` (**required**: `terraform` | `cloudformation` | `sam`)- Which branch of the gate to run.  
- `working_directory` (string, default `.`)- Path to IaC (e.g., `terraform`).  
- `aws_region` (string, default `us-east-1`)- AWS region for operations.  
- `aws_role_arn` (string)- OIDC role to assume; required for planning/applying against AWS or remote TF backend.  
- `apply` (boolean, default `false`) – If `true`, performs apply/execute/deploy (generally env-gated).  
- `environment` (string, default `prod`) – GitHub Environment name (for approvals & env secrets).  
- `terraform_version` (string, default `1.14.0`)- Terraform version.  
- `cfn_template` (default `template.yaml`)- CFN/SAM template path.  
- `cfn_stack_name` (string, optional)- Stack name for CFN/SAM.  
- `cfn_capabilities` (string, default `CAPABILITY_NAMED_IAM`)- CFN capabilities.  
- `cfn_parameter_overrides` (string, optional)- Space-separated `KEY=VALUE` pairs for CFN.  
- `sam_config` (string, default `samconfig.toml`)- Optional SAM config path.  
- `enable_checkov` (boolean, default `true`)- Run Checkov security/policy checks using a large built-in ruleset.  
- `enable_conftest` (boolean, default `true`)- Run Conftest policy checks.  
- `policy_dir` (string, default `policy`)- Directory with Rego policies for Conftest (used if present).  

## How to Use (caller repos)  
Replace inputs such as `<your-org>/<this-repo>` and `aws_role_arn` with your values. Pin to a **tag** (e.g., `v1`) or a **commit SHA** for reproducibility.  

### CloudFormation  
```yaml
name: infra-ci
on:
  push:
    branches: [ main ]
    paths:
      - 'template.yaml'
      - '.github/workflows/infra-ci.yml'
  workflow_dispatch: {}
permissions:
  id-token: write
  contents: read

jobs:
  gate:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: cloudformation
      cfn_template: template.yaml
      cfn_stack_name: my-stack
      cfn_capabilities: CAPABILITY_NAMED_IAM
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/GitHubOIDC-Infra
      environment: prod
      apply: false
```      

### SAM  
```yaml
name: infra-ci
on:
  push:
    branches: [ main ]
    paths:
      - 'template.yaml'
      - 'samconfig.toml'
      - '.github/workflows/infra-ci.yml'
  workflow_dispatch: {}
permissions:
  id-token: write
  contents: read

jobs:
  gate:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: sam
      cfn_template: template.yaml
      cfn_stack_name: my-sam-stack
      sam_config: samconfig.toml
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/GitHubOIDC-Infra
      environment: prod
      apply: true
```

### Terraform  
```yaml
name: infra-ci
on:
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'
      - '.github/workflows/infra-ci.yml'
  workflow_dispatch: {}
permissions:
  id-token: write
  contents: read

jobs:
  gate:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: terraform
      working_directory: terraform
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/GitHubOIDC-Infra
      environment: prod
      apply: false   # set true to deploy (approve in the Environment)
```

## Permissions & Prerequisites  
- **GitHub → AWS OIDC:** create an AWS IAM role trusted to `token.actions.githubusercontent.com` with a `sub` condition restricting to the **caller repo/branch**. Pass the role ARN via `aws_role_arn`.  
- **Terraform remote backend** (if used): ensure the resources already exist, and grant the OIDC role access to them.  
- **Least privilege:** scope each role to only the AWS services your IaC manages (e.g., CloudFront, S3, API Gateway, Lambda, DynamoDB, Route 53, WAF, etc.).  
- **No long-lived keys** in GitHub; OIDC only.  

## Conftest Policies (examples)  
> The gate runs Conftest only when `enable_conftest: true` **and** the caller repo contains `policy/` with Rego files. These are minimal examples. Expand them to fit your standards (e.g., S3 bucket policies, SSE/KMS, WAF association, logging, etc.).  

### CloudFormation / SAM (template file)  
`policy/cfn_s3.rego`
```rego
package cfn.s3

deny[msg] {
  resname := k
  res := input.Resources[resname]
  res.Type == "AWS::S3::Bucket"
  ac := res.Properties.AccessControl
  ac == "PublicRead" or ac == "PublicReadWrite"
  msg := sprintf("S3 bucket %q has public AccessControl (%s)", [resname, ac])
}
```

`policy/cfn_cloudfront.rego`
```rego
package cfn.cloudfront

deny[msg] {
  resname := k
  res := input.Resources[resname]
  res.Type == "AWS::CloudFront::Distribution"
  vp := res.Properties.DistributionConfig.DefaultCacheBehavior.ViewerProtocolPolicy
  not (vp == "redirect-to-https" or vp == "https-only")
  msg := sprintf("CloudFront %q must enforce HTTPS (ViewerProtocolPolicy)", [resname])
}
```

### Terraform (tfplan.json)  
`policy/s3.rego`
```rego
package terraform.s3

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket"
  after := rc.change.after
  after.acl == "public-read" || after.acl == "public-read-write"
  msg := sprintf("S3 bucket %q uses public ACL (%s)", [after.bucket, after.acl])
}
```

`policy/cloudfront.rego`
```rego
package terraform.cloudfront

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_cloudfront_distribution"
  after := rc.change.after
  vp := after.default_cache_behavior.viewer_protocol_policy
  not (vp == "redirect-to-https" or vp == "https-only")
  msg := "CloudFront must enforce HTTPS (viewer_protocol_policy)"
}
```

## Artifacts  
- **Terraform:** `tfplan.binary` and `tfplan.json`  
- **CloudFormation:** `changeset.json` (details of the created change set)  
- **SAM:** `.aws-sam/**` (build output) and `changeset.json` (if created)  

## Versioning  
- Tag releases (e.g., v1.0.0) and optionally maintain a moving major tag (`v1`) for compatible updates.  
- **Recommended for reproducibility:** callers pin to a **commit SHA**.  
Example:
```yaml
uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@<commit-sha>
```

## License  
This project is licensed under the [MIT License](LICENSE).  

---

## Author  
**Patrick Heese**  
Cloud Administrator | Aspiring Cloud Engineer/Architect  
[LinkedIn Profile](https://www.linkedin.com/in/patrick-heese/) | [GitHub Profile](https://github.com/patrick-heese)  
