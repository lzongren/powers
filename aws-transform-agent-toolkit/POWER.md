---
name: "aws-transform-agent-toolkit"
displayName: "AWS Transform Agent Toolkit"
description: "Build agents to run in AWS Transform. This power provides a self-service agent lifecycle from inception to development to production. Build modernization and migration agents with citation-backed AWS Transform documentation search, package agents as containers, deploy to Bedrock AgentCore across platforms (Windows/macOS/Linux), and register with AWS Transform."
author: "AWS"
keywords: ["aws transform", "agent development", "composability", "modernization", "migration"]
---

## Onboarding

### Step 1: Validate tools and access

Before using this power, ensure the following are installed and configured:

- **Python 3.11+**: Required for AWS Transform Agent SDK
  - Verify with: `python3 --version`
  - **CRITICAL**: Python 3.11 or higher is required. The SDK will not work with earlier versions.

- **AWS CLI**: Required for deploying to Bedrock AgentCore and accessing AWS Transform registry
  - Verify with: `aws --version`
  - **CRITICAL**: Must be configured with credentials that have access to your AWS account.
  - Test access: `aws sts get-caller-identity`

- **Finch or Docker**: Required for building ARM64 container images
  - Verify Finch: `finch --version`
  - Or Docker: `docker --version`
  - **CRITICAL**: Container runtime must be running before building images.

- **AWS Transform Account Allowlisting**: Your AWS account must be allowlisted by AWS Transform team if you want to publish to registry.
  - **CRITICAL**: Contact your Solutions Architect to request allowlisting before proceeding with agent registration.
  - Without allowlisting, agent registration will fail.

### Step 2: Install AWS Transform Agent SDK

Install the SDK from PyPI into a virtual environment:

```bash
cd <user-project>
python3 -m venv .venv && source .venv/bin/activate
pip install agent-builder-sdk-aws-transform \
    agent-builder-agentic-mcp-aws-transform \
    agent-builder-types-aws-transform \
    agent-builder-mcp-client-aws-transform
```

Windows PowerShell:
```powershell
cd <user-project>
py -3 -m venv .venv; .venv\Scripts\Activate.ps1
pip install agent-builder-sdk-aws-transform `
    agent-builder-agentic-mcp-aws-transform `
    agent-builder-types-aws-transform `
    agent-builder-mcp-client-aws-transform
```

**Verify installation:**

```bash
python3 -c "import agent_builder_sdk; print('SDK OK')"
```

**Register botocore service models:**

The SDK ships with custom botocore service models that must be registered before use:

macOS/Linux:
```bash
SDK_MODELS=$(python3 -c "from importlib.resources import files; print(files('agent_builder_sdk').joinpath('botocore_models'))")
aws configure add-model --service-name atxagentregistryexternal \
  --service-model "file://${SDK_MODELS}/atxagentregistryexternal/2022-07-26/service-2.json"
aws configure add-model --service-name transformagenticservice \
  --service-model "file://${SDK_MODELS}/transformagenticservice/2018-05-10/service-2.json"
```

Windows PowerShell:
```powershell
$SDK_MODELS = python3 -c "from importlib.resources import files; print(files('agent_builder_sdk').joinpath('botocore_models'))"
aws configure add-model --service-name atxagentregistryexternal --service-model "file://$SDK_MODELS\atxagentregistryexternal\2022-07-26\service-2.json"
aws configure add-model --service-name transformagenticservice --service-model "file://$SDK_MODELS\transformagenticservice\2018-05-10\service-2.json"
```

**CRITICAL**: Without these models, the SDK will fail at runtime with `Unknown service: 'transformagenticservice'`.

### Step 3: Set up IAM roles

AWS Transform agent deployment requires two IAM roles in your AWS account:

- **`AgentCoreExecutionRole`** — used by Bedrock AgentCore to run your agent container. Needs Bedrock model access, `transform-agents:*`, ECR pull, CloudWatch Logs, and X-Ray permissions.
- **`AWSTransformAgentInvokeRole`** — assumed by the AWS Transform compute service to invoke your Bedrock AgentCore runtime. Needs `bedrock-agentcore:InvokeAgentRuntime`, `GetAgentRuntime`, and `GetAgentRuntimeEndpoint`.

**Missing or incorrectly configured roles are the single most common cause of silent deployment failures** — the runtime reaches READY but jobs fail ~8 minutes after creation with "Failed to start the job" in the AWS Transform webapp.

**Check if the roles already exist:**

```bash
aws iam get-role --role-name AgentCoreExecutionRole --query 'Role.Arn' --output text
aws iam get-role --role-name AWSTransformAgentInvokeRole --query 'Role.Arn' --output text
```

If either command returns `NoSuchEntity`, you need to create the roles.

**Create both roles using the provided CloudFormation template:**

A complete, correct CloudFormation template with both roles, all required permissions, and the right trust policies is provided in [steering/deployment-pipeline-guide.md](./steering/deployment-pipeline-guide.md#section-2-complete-cloudformation-template).

Save the template as `iam-roles.yaml` and deploy it:

```bash
aws cloudformation deploy \
  --template-file iam-roles.yaml \
  --stack-name aws-transform-agent-iam-roles \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**CRITICAL — Trust policy:** `AWSTransformAgentInvokeRole` MUST trust `prod.us-east-1.compute.elastic-gumby.aws.internal`. The template handles this correctly.

**Regional scope:** The AWS Transform Compute principal format is `prod.{region}.compute.elastic-gumby.aws.internal`, and AWS Transform is available in several regions. This power, its CloudFormation template, and the deployment tooling assume us-east-1 only. Using a non us-east-1 AWS Transform region requires swapping the region segment in both principals, pointing the registry endpoint at the matching region, and passing `region` explicitly to `deploy_agent_full_pipeline`.

**If your roles have non-default names** (e.g., set up via the AWS console or Bedrock AgentCore SDK which creates roles like `AmazonBedrockAgentCoreSDKRuntime-...`):

- `AgentCoreExecutionRole`: the MCP deployment tool first tries the default name, then falls back to scanning trust policies for a role trusting `bedrock-agentcore.amazonaws.com`. If exactly one match is found it's used automatically; if zero or multiple, you'll get an error asking you to pass `execution_role_arn` explicitly.
- `AWSTransformAgentInvokeRole`: the MCP deployment tool only looks up the exact default name. If your invoke role has a different name, you must pass `access_role_arn` explicitly — otherwise registry registration is skipped with a warning.

For the complete permissions reference for both roles, see [steering/deployment-pipeline-guide.md](./steering/deployment-pipeline-guide.md#section-1-iam-roles-overview).

### Step 4: Add workspace hooks

Add a hook to validate deployment prerequisites before deploying agents:

`.kiro/hooks/validate-deployment.kiro.hook`
```json
{
  "enabled": true,
  "name": "Validate AWS Transform Deployment Prerequisites",
  "description": "Check IAM roles, container runtime, and AWS access before deployment",
  "version": "1",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "Before deploying AWS Transform agents, verify: 1) AWS credentials are valid (aws sts get-caller-identity), 2) finch or docker is running, 3) IAM roles exist and have correct permissions (AgentCoreExecutionRole with bedrock:InvokeModel and transform-agents:*; AWSTransformAgentInvokeRole with bedrock-agentcore:InvokeAgentRuntime). Report any missing prerequisites."
  }
}
```

This hook helps catch common deployment issues before they cause failures in the pipeline.

### Step 5: Configure MCP Server Environment (Optional)

The MCP server is installed via `uvx` (see `mcp.json`). To pass environment variables, edit the `mcp.json` in your power directory:

```json
{
  "mcpServers": {
    "aws-transform-agent-toolkit": {
      "command": "uvx",
      "args": ["agent-builder-mcp-aws-transform"],
      "env": {
        "AWS_PROFILE": "my-profile",
        "AWS_REGION": "us-east-1"
      }
    }
  }
}
```

**Environment variables:**
- `AWS_PROFILE`: AWS CLI profile to use for credentials (set via `aws configure` or `aws sso login`)
- `AWS_REGION`: AWS region for the AWS Transform (defaults to us-east-1)

Restart Kiro after making changes.

## Deployment Automation

Kiro can generate complete deployment pipelines covering:
- Docker image building (with SDK and MCP runtime)
- ECR repository setup and image push
- Bedrock AgentCore runtime creation with `bedrock-agentcore-control`
- AWS Transform agent registration with correct API parameters
- IAM role CloudFormation templates

See [steering/deployment-pipeline-guide.md](./steering/deployment-pipeline-guide.md) for detailed patterns and best practices.

**Example reference implementation:** See the `pipeline/` directory in the AWS Transform demo project for a working deployment pipeline.

**Platform Compatibility:**
- **Windows**: Uses AWS CodeBuild for container builds (finch not available)
- **macOS/Linux**: Uses finch or docker for local builds
- **All platforms**: MCP deployment tools automatically detect best approach

For conversational deployment workflow, see [steering/deploy-agent-workflow.md](./steering/deploy-agent-workflow.md).

## MCP Tools Available

This power includes an MCP server with search and registration tools:

### Search Tools
- **keyword_search(query, top_k)** - Search AWS Transform documentation using keyword matching (recommended)
- **search_by_source(query, source, top_k)** - Search filtered by source
  (`dev-guide`, `sdk`, `agentic-api`, `registry-api`, and the documented
  HITL-specific sources)

### Agent Deployment Tools
- **build_agent_image** - Build AWS Transform agent Docker image for ARM64 platform
  - Supports three build methods: local finch, local Docker, or AWS CodeBuild (required for Windows)
  - Automatically detects best runtime for current platform
  - Pushes image to ECR (creates repository if needed)
- **deploy_agent_to_agentcore** - Deploy agent image to Bedrock AgentCore
  - Creates Bedrock AgentCore runtime and polls until READY
  - Generates unique runtime names with timestamp to avoid conflicts
- **deploy_agent_full_pipeline** - Complete deployment pipeline: build → push → deploy → register
  - Orchestrates all phases for full agent deployment to AWS Transform
  - Auto-detects IAM roles (AgentCoreExecutionRole, AWSTransformAgentInvokeRole)
  - Platform-aware: Windows users automatically use CodeBuild, macOS/Linux prefer local finch/Docker

### Agent Registry Tools
- **register_agent** - Register and publish a new agent with AWS Transform Agent Registry
  - Performs all three registration steps: RegisterAgent → PublishAgentVersion → UpdatePublisherAccessControl
  - Use after deploying your agent to Bedrock AgentCore to register it with AWS Transform
- **get_agent** - Get details of a registered agent
- **get_agent_version** - Get a specific version of a registered agent
- **update_agent** - Update an existing agent's metadata
- **list_agents_by_publisher** - List all agents published by the current account
- **publish_agent_version** - Publish a new version of an existing agent
  - Copies config from the current (or specified) version, applies optional overrides (runtimeArn, atxAccessRoleArn), and publishes the new version
  - Use after initial registration to iterate on agent versions
- **list_agent_access_control** - List access control settings for an agent
- **update_publisher_access_control** - Grant or revoke account access to an agent

### Debugging Tools
- **fetch_logs** - Fetch CloudWatch logs for an agent runtime
- **list_log_streams** - List available log streams for an agent runtime
- **validate_agent_setup** - Validate agent deployment prerequisites (IAM roles, ECR, etc.)

### HITL Tools
- **get_hitl_generation_prompt** - Get the full HITL UI generation rules and component schema

The MCP server provides access to:
- AWS Transform Developer Guide (architecture, workflows, testing)
- BaseAgent SDK documentation (AsyncBaseOrchestrator, AsyncBaseSubagent)
- Agentic API and Agent Registry API specifications

## Verification Guidelines

**FIRST STEP FOR EVERY AWS TRANSFORM QUESTION: call `keyword_search` (or
`search_by_source`) at least once before you answer — including how-to,
troubleshooting, "what am I missing", "which role", and "what should it trust"
questions.** This is mandatory whenever the tools are loaded, even if you already
know the answer or a steering file obviously contains it. Reading a `steering/*.md`
file is NOT a substitute for calling search; it is an additional step you may take
AFTER searching to add depth. Do not answer from a file read alone when the search
tools are available.

When answering AWS Transform questions:
1. **Search first, always** - Call `keyword_search`/`search_by_source` before answering
2. **Verify CLI commands** - Use `keyword_search("aws cli")` to confirm correct service names
3. **Verify API names** - Use `search_by_source(query, "agentic-api")` or
   `search_by_source(query, "registry-api")`, as appropriate, to confirm operation
   names
4. **Cross-reference** - Check steering files for patterns AFTER the MCP search

Never guess: CLI commands, API operation names, service endpoints, or registration steps.

### If search tools are genuinely unavailable (fallback only)

This fallback applies ONLY when `keyword_search` / `search_by_source` are truly not
loaded in this session (calling them errors or they do not exist). It is NOT a
shortcut to skip search when the tools ARE available. In that genuine-unavailability
case, do NOT answer from memory — instead ground every answer by reading the bundled
`steering/*.md` files, installed SDK, or bundled service models directly, using the
SAME typed citation tags (`[api:...]`, `[sdk:...]`, `[dev-guide:...]`).

- **Read the FULL relevant section, not just the top of a file.** Steering docs are
  long; the answer often lives deep in the file. Search within the file for the
  relevant term, then read the surrounding section.
- **Inspect authoritative SDK or service-model definitions for exact details.**
  Locate the installed package, then read the complete class, method, operation, or
  shape rather than relying on a partial match.

### Search → Read → Generate (IMPORTANT)

Search results are **truncated previews** (500 chars). Before generating code from a search result, read the full source to get complete signatures, parameters, and implementation details.

When a search result includes a `file` field (e.g., `"file": "agent_builder_sdk/orchestrator.py"`):

1. **Find** the installed package location:
   ```bash
   python3 -c "import agent_builder_sdk; print(agent_builder_sdk.__file__)"
   ```
2. **Grep** for the class or function in that location:
   ```bash
   grep -r "class BaseOrchestrator" $(python3 -c "import agent_builder_sdk, os; print(os.path.dirname(agent_builder_sdk.__file__))")
   ```
3. **Read** the matched file for full signatures and docstrings
4. **Generate** code using the complete source — not the truncated preview

**When you cite something found by grepping or reading a file directly, still emit a
typed tag, never the raw path.** Map an SDK class/function to `[sdk:<Symbol>]`, an API
operation to `[api:<Operation>]`, and a steering document to
`[dev-guide:<topic>]`. For example, after reading `agent_runtime_server.py`, cite
`[sdk:AgentRuntimeServer]`, not the file path.

This avoids hardcoding the Python version in the site-packages path. The `file` field
tells you which file contains the code; the class or function name from the result
tells you what to search for; the OS-specific SDK install path tells you where.

## Grounding Rules (CRITICAL)

**ALWAYS cite your sources.** Every AWS Transform-specific factual claim must include
typed citation tags from the search results or directly inspected authoritative
source used to ground it.

**Whenever you name an SDK symbol, API operation, or documentation topic — whether it
came from a search card, grep, or file read — attach its typed tag
(`[sdk:Symbol]`, `[api:Operation]`, `[dev-guide:topic]`). A bare file path such as
`queue_handler.py` is NEVER a citation.** When listing real methods of a class, each
method named must carry its own `[sdk:...]` tag.

1. **NEVER answer from memory** - Always search first using MCP tools
2. **If not found, say so** - Respond with "I don't have information about X in the
   AWS Transform documentation". Every no-results response MUST also end with at
   least one documentation-discovery next step: check the Developer Guide directly,
   rephrase the query, or contact your Solutions Architect. This discovery suggestion
   is required even if you also offer an implementation workaround (for example, a
   custom `@tool`); the workaround does not replace it.
3. **Cite AWS Transform-specific claims in EVERY response** using typed citation tags:
   - Format: `[type:name]`, e.g., `[sdk:AsyncBaseOrchestrator]`,
     `[api:RegisterAgent]`, `[dev-guide:doc]`
   - **The tag type is determined by WHAT you cite, never by which tool produced
     it.** Use the same grammar for search results and directly inspected files:
     - SDK class/function/method → `[sdk:<Symbol>]` (e.g., `[sdk:AsyncBaseSubagent]`)
     - API operation → `[api:<Operation>]` (e.g., `[api:RegisterAgent]`)
     - Steering document / Developer Guide topic → `[dev-guide:<topic>]` (e.g., `[dev-guide:agent-registration]`)
   - **NEVER cite a raw file path** (e.g.,
     `[source: steering/agent-registration.md]`). Map the file to its typed tag
     instead.
   - **NEVER echo the MCP `source` field as the tag namespace.** Normalize:
     - `AgentBuilderSDK`, `TransformHITLComponentPythonSDK`, and
       `TransformHITLComponentJavaSDK` → `[sdk:...]`
     - `TransformAgenticApiModel` and `AWSTransformAgentRegistryExternalServiceModel` → `[api:...]`
     - `AWSTransform-Developer-Guide` and `HITL-*` documentation sources → `[dev-guide:...]`
   - **In tables or lists of methods/APIs, put the typed tag inline in each row.** A
     "source file" column with a raw filename does NOT count as a citation.
   - **Worked example:** write
     `AsyncBaseOrchestrator.process_message_async [sdk:AsyncBaseOrchestrator.process_message_async]`
     and `AgentRuntimeServer.handle_stop [sdk:AgentRuntimeServer.handle_stop]` — NOT
     `process_message_async (agent_runtime_server.py)`.
   - Place citations inline or at the end of relevant statements
   - If multiple sources, cite all of them
4. **Low confidence = search again** - Try different queries before guessing
5. **Consolidate code snippets** - When search returns code examples:
   - Verify API operations: `search_by_source("OperationName", "agentic-api")` or
     `search_by_source("OperationName", "registry-api")`
   - Verify SDK classes: `search_by_source("ClassName", "sdk")`
6. **Iterate if needed** - If first search results are insufficient:
   - Start broad: `keyword_search("orchestrator")`
   - Then narrow: `keyword_search("orchestrator invoke subagent")`
   - Evaluate results before answering - if low relevance, search again
7. **Low-score results** — If top result BM25 score is very low or results seem
   irrelevant, try: (a) different terminology, (b) `search_by_source` to narrow scope,
   (c) read the SDK source directly via filesystem. Do NOT generate code from
   low-confidence search results.

Example workflow:
1. User asks about agent registration
2. Call `keyword_search("agent registration")` or `search_by_source("RegisterAgent", "registry-api")`
3. If results found → Answer using ONLY retrieved content + include citation tag from results
4. If not found → Say "I don't have this in the indexed docs" and suggest checking
   the Developer Guide or contacting your SA

### Scope and Fabrication Boundaries

These topics have operational impact, so fabricated details can cause failures:

- **IAM role names and ARN formats:** Use exact names from the docs
  (`AgentCoreExecutionRole`, `AWSTransformAgentInvokeRole`). Do not infer custom role
  name patterns.
- **API endpoint URLs:** Search for the exact endpoint in the documentation. Do not
  construct URLs from partial patterns.
- **CLI service names:** Search the appropriate API source to verify them. Do not
  guess based on AWS naming patterns.
- **Undocumented third-party integrations** (Salesforce, Stripe, Slack, or any other
  external SaaS/API not in the docs): after the honest "I don't have information
  about X" disclaimer, describe ONLY the AWS Transform-native extension mechanism in
  the abstract (custom `@tool`, `custom_tools`, connector schema, MCP client) and make
  the third-party client the user's responsibility. A disclaimer followed by an
  invented implementation is still fabrication. Do NOT emit service-specific code,
  concrete method names, pinned library versions, or auth flows, and NEVER attach AWS
  Transform citation tags to invented third-party code.

**Carve-out — standard AWS-service configuration is NOT fabrication.** The "never
answer from memory" rule protects AWS Transform-specific facts (role names, ARN
formats, API operations, endpoint URLs, and naming conventions), which must come from
search. It does not bar you from acknowledging standard configuration categories of
an AWS service the user asks about, such as ECR image scanning, KMS encryption,
lifecycle policies, CloudWatch retention, or S3 versioning. When the docs return
nothing AWS Transform-specific: (1) state plainly that AWS Transform does not require
or specify special settings, then (2) briefly name the standard AWS options, clearly
framed as standard service features rather than AWS Transform requirements. Do not
attach AWS Transform citation tags to this general-knowledge portion.

### IAM Role Boundary

- `AgentCoreExecutionRole` is the role the Bedrock AgentCore runtime uses for the
  agent container's AWS API calls.
- `AWSTransformAgentInvokeRole` is the role AWS Transform assumes to invoke that
  runtime.
- For session-duration questions, first identify which credential is expiring. If AWS
  credentials expire inside the running container, focus on
  `AgentCoreExecutionRole` and the container's credential refresh behavior;
  `AWSTransformAgentInvokeRole` is not the source of in-container credentials. State
  the relevant role clearly instead of suggesting that both roles need changes.

# When to Load Steering Files

**Load a steering file only AFTER calling `keyword_search`/`search_by_source` for the
question.** Search is the mandatory first step; the files below provide additional
depth and are never a way to skip search.

- Getting started with AWS Transform or building your first agent → `steering/getting-started.md`
- Building a new agent from scratch (orchestrator or subagent) → `steering/orchestrator-patterns.md` or `steering/subagent-patterns.md`
- Creating or modifying orchestrator agents → `steering/orchestrator-patterns.md`
- Building or updating subagents → `steering/subagent-patterns.md`
- Working with AWS Transform APIs (Agentic API, Registry API) → `steering/api-reference.md`
- Registering agents, publishing versions, understanding agentCard schema for composability → `steering/agent-registration.md`
- Deploying agents (Docker, ECR, Bedrock AgentCore, pipeline automation) → `steering/deployment-pipeline-guide.md`
- Working with the skill registry (upload, download, share, manage agent skills) → `steering/skill-operations.md`
  - Skills are reusable capabilities that expand what an agent can do. The skill registry is a central repository where developers can choose from or contribute to a bank of skills to use with their agents.
- Troubleshooting agent deployment or runtime issues → `steering/troubleshooting.md`
- Adding an agent to an existing workflow → `steering/workflow-integration.md`
- **Deploying agents through Kiro (recommended workflow)** → `steering/deploy-agent-workflow.md`

## Next Steps

1. Read the architecture overview in `getting-started.md`
2. Follow the patterns in `orchestrator-patterns.md` or `subagent-patterns.md`
3. Use the inline code examples to scaffold your agent, then customize
4. Test locally before deploying to Bedrock AgentCore

## License
AWS Service Terms. This power is provided by AWS and is subject to the AWS Customer Agreement and applicable AWS service terms.

This power integrates with [agent-builder-mcp-aws-transform](https://github.com/awslabs/agent-builder-toolkit-aws-transform) (Apache-2.0 license).
