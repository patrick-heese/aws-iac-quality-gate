# Reusable IaC Quality Gate (GitHub Actions)  
This repository provides a single reusable GitHub Actions workflow that standardizes **IaC quality checks** across repos. It runs linting, produces plans/change sets as artifacts, and can optionally apply behind GitHub Environment approvals using OIDC-assumed AWS roles.  
- **CloudFormation**: **cfn-lint**, **Change Set** creation (no-execute), artifact upload, optional **execute** (env-gated).  
- **SAM**: `sam validate --lint`, `sam build`, **Change Set** creation (no-execute), artifact upload, optional **deploy** (env-gated).  
- **Terraform**: `fmt`, `validate`, **TFLint**, **plan (JSON)**, artifact upload, optional **apply** (guarded by environment approvals).  

## Project Structure  
```plaintext
aws-iac-quality-gate
├── .github/workflows/            # GitHub Actions workflow
│   └── reusable-iac-gate.yml     # IaC Quality Gate
├── LICENSE                       
└── README.md                     
```

## How it Works  
- Caller repos **invoke** this workflow using `workflow_call`.  
- The gate runs tool-specific **linting + policy checks**, produces **plans/changesets** as artifacts, and (optionally) **applies** behind a protected GitHub **Environment** (e.g., `prod`) via **OIDC-assumed** AWS roles.  
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
- `tf_backend_bucket` (string)- S3 bucket for Terraform backend (if used).  
- `tf_backend_bucket` (string)- Object key for Terraform state (if used).  
- `tf_backend_use_lockfile` (boolean, default `false`)- Use S3 native lock file (.tflock) (if used).  
- `tf_workspace` (string)- Terraform workspace name (optional).  
- `cfn_template` (default `template.yaml`)- CFN/SAM template path.  
- `cfn_stack_name` (string, optional)- Stack name for CFN/SAM.  
- `cfn_capabilities` (string, default `CAPABILITY_NAMED_IAM`)- CFN capabilities.  
- `cfn_parameter_overrides` (string, optional)- Space-separated `KEY=VALUE` pairs for CFN.  
- `sam_config` (string, default `samconfig.toml`)- Optional SAM config path.  

## How to Use (caller repos)  
Replace inputs such as `<your-org>/<this-repo>` and `aws_role_arn` with your values. Pin to a **tag** (e.g., `v1`) or a **commit SHA** for reproducibility.  

### CloudFormation  
```yaml
name: infra-ci
on:
  push:
    branches: [ main ]
    paths:
      - 'cloudformation/**'
      - '.github/workflows/infra-ci.yml'
  workflow_dispatch: {}
permissions:
  id-token: write
  contents: read

concurrency:
  group: infra-ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  gate-plan:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: cloudformation
      cfn_template: cloudformation/template.yaml
      cfn_stack_name: cfn-stack-${{ github.run_id }}
      cfn_capabilities: CAPABILITY_NAMED_IAM
      cfn_parameter_overrides: file://cloudformation/params.json
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
      environment: prod  # Remove if no env approvals and apply = "false"
      apply: false 
```      

### SAM  
```yaml
name: infra-ci
on:
  push:
    branches: [ main ]
    paths:
      - 'cloudformation/**'
      - '.github/workflows/infra-ci.yml'
  workflow_dispatch: {}
permissions:
  id-token: write
  contents: read

jobs:
  gate-plan:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: sam
      cfn_template: cloudformation/template.yaml
      cfn_stack_name: sam-stack-${{ github.run_id }}
      sam_config: cloudformation/samconfig.toml
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
      environment: prod  # Remove if no env approvals and apply = "false"
      apply: false 
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
  contents: read
  id-token: write

concurrency:
  group: infra-ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  gate:
    uses: <your-org>/<this-repo>/.github/workflows/reusable-iac-gate.yml@v1
    with:
      iac_tool: terraform
      working_directory: terraform
      aws_region: us-east-1
      aws_role_arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
      environment: prod  # Remove if no env approvals and apply = "false"
      apply: false
```

## Permissions & Prerequisites  
- **GitHub → AWS OIDC:** Create an AWS IAM role trusted to `token.actions.githubusercontent.com` with a `sub` condition restricting to the **caller repo/branch**. Pass the role ARN via `aws_role_arn`.  
- **Terraform remote backend** (if used): Ensure the resources already exist, and grant the OIDC role access to them.  
- **Least privilege:** Scope each role to only the AWS services your IaC manages (e.g., CloudFront, S3, API Gateway, Lambda, DynamoDB, Route 53, WAF, etc.).  
- **No long-lived keys** in GitHub; OIDC only.  

## License  
This project is licensed under the [MIT License](LICENSE).  

---

## Author  
**Patrick Heese**  
Cloud Administrator | Aspiring Cloud Engineer/Architect  
[LinkedIn Profile](https://www.linkedin.com/in/patrick-heese/) | [GitHub Profile](https://github.com/patrick-heese)  
