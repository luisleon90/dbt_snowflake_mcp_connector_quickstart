# dbt Semantic Layer + Snowflake Cortex Agents via MCP

## What this guide covers

This guide walks you through connecting Snowflake Cortex Agents to the dbt Semantic Layer using the dbt MCP Server and Snowflake's MCP Connector functionality. By the end, a Snowflake Cortex Agent will be able to answer natural language questions about your data — powered entirely by the context your dbt project provides. No Snowflake Semantic View required.

The setup has five stages:

1. Configure the dbt Semantic Layer
2. Configure the dbt MCP Server
3. Configure OAuth in dbt
4. Set up the MCP Connector in Snowflake
5. Create and configure a Cortex Agent

---

## Concepts

### What is the dbt Semantic Layer?

The dbt Semantic Layer is a centralized place to define your business metrics — things like revenue, retention rate, or active users — directly in your dbt project. Instead of different teams writing different SQL to calculate the same number, you define a metric once in dbt and any downstream tool that connects to the Semantic Layer gets a consistent, governed result.

Metrics in the dbt Semantic Layer are defined using YAML files in your dbt project and powered by MetricFlow, dbt's metric framework. When a tool queries the Semantic Layer, dbt handles the SQL generation automatically.

### What is the dbt MCP Server?

The dbt MCP Server exposes the full context of your dbt project over the Model Context Protocol (MCP) — an open standard that allows AI tools to discover and invoke capabilities from external systems. By connecting an AI tool to the dbt MCP Server, it gains access to everything dbt knows about your data: metrics and semantic definitions, model lineage, column-level documentation, data quality test results, and project structure. The AI uses this context to answer questions accurately, with dbt handling the translation to SQL.

The remote dbt MCP Server is hosted by dbt platform and requires no local installation. It is ideal for consumption-based use cases like querying metrics and exploring metadata.

### What is a Snowflake MCP Connector?

An MCP Connector in Snowflake is a Snowflake object that connects Cortex Agents and Snowflake Intelligence to a remote MCP server. It consists of two parts:

- An **API Integration** — stores the server URL and OAuth configuration
- An **External MCP Server** — references the API integration and registers the connection for use by agents

Once created, agents can discover and invoke tools from the MCP server directly within Snowflake's governed environment.

---

## Prerequisites

Before starting, you will need:

- A dbt platform account (Starter or higher) with AI features enabled
- A Snowflake account with ACCOUNTADMIN access or equivalent privileges to create API integrations and external MCP servers
- At least one dbt project with models deployed to a production environment
- The dbt Semantic Layer configured for your project (see Step 1 below)

> **⚠️ Plan requirement:** OAuth-based authentication for the remote dbt MCP Server may be gated to Enterprise or Enterprise+ plans. If you are on a lower tier, confirm with your dbt account team before proceeding. Service token and PAT authentication are available on lower-tier plans.

---

## Step 1: Configure the dbt Semantic Layer

The dbt Semantic Layer must be set up and pointing at a production environment before the MCP Server can expose it. If you already have the Semantic Layer configured, skip to Step 2.

### 1.1 Define semantic models and metrics

Semantic models and metrics are defined as YAML files in your dbt project, stored alongside your models. A semantic model describes the structure of a mart or entity — its dimensions, entities, and measures. A metric references a semantic model and defines a specific business calculation.

**Option A: Define manually**

Create YAML files in your `models/` directory. For example:

```yaml
semantic_models:
  - name: orders
    model: ref('fct_orders')
    entities:
      - name: order_id
        type: primary
    dimensions:
      - name: order_date
        type: time
        type_params:
          time_granularity: day
    measures:
      - name: order_total
        agg: sum
        expr: amount

metrics:
  - name: order_total
    type: simple
    label: "Order Total"
    type_params:
      measure:
        name: order_total
```

For full YAML syntax and metric types, refer to the [dbt Semantic Layer documentation](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl).

**Option B: Use dbt Wizard**

In dbt platform Studio, open the **dbt Wizard** panel and ask it to generate semantic models and metrics for one of your mart models. For example:

> *"Generate a semantic model and metrics for the fct_orders model, including a revenue metric and an order count metric."*

dbt Wizard understands your project structure and will generate syntactically correct YAML that follows dbt conventions.

### 1.2 Run a production job

After defining your semantic models and metrics, run a production dbt job that includes `dbt build` or at minimum `dbt parse`. This generates the `semantic_manifest.json` artifact that the Semantic Layer APIs consume.

1. In dbt platform, navigate to **Orchestration → Jobs**
2. Select your production job and click **Run Now**
3. Wait for the run to complete successfully

> **Note:** The Semantic Layer APIs pull from the most recent manifest produced by a successful production run. If your metrics are not showing up downstream, confirm that the job has run successfully after your metric definitions were committed.

### 1.3 Enable the Semantic Layer

1. In dbt platform, navigate to **Account Settings → Projects** and select your project
2. In the **Project details** page, scroll to the **Semantic Layer** section and click **Configure Semantic Layer**
3. Select the deployment environment you want to run the Semantic Layer against and click **Save**

### 1.4 Generate a service token

The dbt MCP Server uses a service token or personal access token (PAT) to authenticate requests.

1. Navigate to **Account Settings → API Tokens → Service Tokens**
2. Click **+ New Token**
3. Name the token (for example, `snowflake-mcp-token`)
4. Set the permissions to **Semantic Layer Only** and **Metadata Only**
5. Click **Save** and copy the token — you will not be able to view it again

> **Note:** If you plan to use the `execute_sql` tool through the MCP Server, you must use a Personal Access Token (PAT) instead. Service tokens do not support `execute_sql`. For metric querying and metadata exploration, a service token is sufficient.

---

## Step 2: Configure the dbt MCP Server

The dbt remote MCP Server is available at your dbt platform host URL. No installation is required.

### 2.1 Locate your MCP Server URL

1. In dbt platform, navigate to **Account Settings**
2. Under **Access URLs**, locate your dbt platform host URL. It will look like:
   - `https://cloud.getdbt.com` for standard accounts
   - `https://ACCOUNT_PREFIX.us1.dbt.com` for multi-cell accounts
3. Your MCP Server URL is: `https://YOUR_DBT_HOST_URL/api/ai/v1/mcp/`

Keep this URL handy — you will use it when creating the Snowflake API Integration in Step 4.

---

## Step 3: Configure OAuth in dbt

Before setting up the MCP Connector in Snowflake, confirm which authentication method you will use. This determines what you need to have ready before Step 4.

| Method | Best for | What you need |
|---|---|---|
| **DCR (Dynamic Client Registration)** | Most users — recommended | Nothing. dbt platform handles OAuth registration automatically |
| **Custom OAuth App** | Environments where DCR is unavailable | OAuth credentials from your identity provider (see below) |

### Option A: DCR (Dynamic Client Registration) — Recommended

DCR is the recommended approach. With DCR, OAuth app registration is handled automatically by dbt platform — no manual credential setup is required on the dbt side. When Snowflake creates the API Integration using DCR, dbt handles the rest of the handshake.

No additional configuration is needed in dbt platform when using DCR.

### Option B: Custom OAuth App

If DCR is not available in your environment, you can configure a custom OAuth app. You will need to register an OAuth application in your identity provider and collect the following values before proceeding to Step 4:

| Item | Description |
|---|---|
| OAuth Client ID | Client ID from your registered OAuth app |
| OAuth Client Secret | Client secret from your registered OAuth app |
| OAuth Token Endpoint | Token exchange endpoint from your identity provider |
| OAuth Authorization Endpoint | Authorization endpoint from your identity provider |
| Scopes | `user_access` and `offline_access` |

---

## Step 4: Set Up the MCP Connector in Snowflake

The MCP Connector in Snowflake consists of two objects: an **API Integration** and an **External MCP Server**. There are two ways to create them.

### What you will need

| Item | Where to find it |
|---|---|
| dbt MCP Server URL | dbt platform → **Account Settings → Access URLs** |
| Snowflake database | Where you want to create the connector object |
| Snowflake schema | Where you want to create the connector object |
| Auth credentials | DCR (no credentials needed) or custom OAuth values from Step 3 |

---

### Option A: Manual setup via Snowsight UI

#### 4a.1 Open the connector settings

1. In Snowsight, navigate to **AI & ML → Cortex Agents**
2. Click **Settings** in the top right
3. Select **Tools and Connectors**
4. Click **Add Custom MCP Connector**

#### 4a.2 Configure the connector

1. Enter a name for your connector — for example, `DBT_MCP_CONNECTOR`
2. In the **MCP Server URL** field, paste your dbt MCP Server URL from Step 2.1
3. Select the **Database** and **Schema** where the connector object will be created
4. Under **Authentication**, select **DCR** as the authentication method (or enter your custom OAuth values if using Option B from Step 3)
5. Click **Add**

> **Note on scopes:** If you encounter errors during authentication of the MCP, ensure that `OAUTH_ALLOWED_SCOPES = ('user_access', 'offline_access')` was included in external access integration. This is a known requirement and the integration will fail without it. The UI may not surface this field explicitly — if configuring via SQL, always include it (see the DDL reference below).

> **Note on network access:** Snowflake requires an External Access Integration to allow network-level access to the dbt MCP Server URL. When configuring manually, your Snowflake account admin may need to ensure this exists before the connector can communicate with the dbt MCP Server. If you use Option B below (Cortex Code), this is handled by the agent.

---

### Option B: Setup via Cortex Code (CoCo) — Recommended

Cortex Code is Snowflake's AI coding assistant, available in Snowsight. When you describe what you want to create, CoCo generates and executes the required SQL — including the External Access Integration and the `OAUTH_ALLOWED_SCOPES` parameter — automatically.

#### 4b.1 Open Cortex Code

1. In Snowsight, open a new **Cortex Code** session

#### 4b.2 Ask CoCo to create the connector

Describe what you want to create. For example:

> *"Create a Custom MCP Connector for the dbt MCP Server. The MCP URL is `https://YOUR_DBT_HOST_URL/api/ai/v1/mcp/`. Create the connector in `YOUR_DATABASE.YOUR_SCHEMA`, use DCR for authentication, and include OAuth allowed scopes of user_access and offline_access."*

Review the SQL CoCo generates — it will include the External Access Integration and the External MCP Server object — then confirm and execute.

> **✅ Expected:** CoCo creates the External Access Integration and the External MCP Server object in your specified schema. You should see both objects created successfully in the output.

---

### 4.3 Authenticate into the MCP Connector

Once the connector object is created, you need to authenticate into it using your dbt platform credentials. This authorizes Snowflake to make requests to the dbt MCP Server on your behalf.

1. In Snowsight, navigate to **AI & ML → Cortex Agents → Settings**
2. Select **MCP Connectors**
3. Locate your connector and click **Connect**
4. Follow the prompts — you will be redirected to dbt platform to complete the OAuth authorization flow
5. Sign in with your dbt platform credentials and grant the requested permissions
6. Return to Snowsight — your connector should now show a **Connected** status

> **✅ Expected:** The connector shows **Connected**. Cortex Agents configured to use this connector can now access the full context dbt provides about your data.

---

## Step 5: Create a Cortex Agent

With the MCP Connector authenticated, you can now create a Cortex Agent that uses it as its data source.

### Option A: Create via Snowsight UI

#### 5a.1 Create a new agent

1. In Snowsight, navigate to **AI & ML → Cortex Agents**
2. Click **+ New Agent**
3. Give your agent a name — for example, `MY_DATA_AGENT`
4. Add a description that reflects what the agent does — for example:

> *"An AI assistant for answering data questions, grounded in the context provided by the dbt Semantic Layer."*

#### 5a.2 Add the MCP Connector as a tool

1. Under **Tools**, click **MCP Connectors**
2. Browse the available connectors and select the connector you created in Step 4
3. Click **Add to agent**

#### 5a.3 Add a system prompt (optional but recommended)

A system prompt guides the agent's behavior and helps it stay focused on the right domain. For example:

> *"You are a data assistant. Answer questions about business metrics and data using the available tools. Be concise, cite specific numbers when possible, and clarify your assumptions when the question is ambiguous."*

4. Click **Save**

> **✅ Expected:** Your agent is created and visible in the Cortex Agents list. It is configured to access the context provided by dbt through the MCP Connector.

---

### Option B: Create via Cortex Code

#### 5b.1 Open Cortex Code

1. In Snowsight, open a new **Cortex Code** session

#### 5b.2 Ask CoCo to create the agent

1. In the CoCo prompt, describe the agent you want to create — for example:

> *"Create a Cortex Agent called MY_DATA_AGENT that uses the `YOUR_DATABASE.YOUR_SCHEMA.DBT_MCP_SERVER` MCP server as a tool. Add a system prompt describing it as a data assistant grounded in the context provided by dbt."*

2. Review the SQL CoCo generates
3. Confirm and execute

> **✅ Expected:** The agent is created and appears in **AI & ML → Cortex Agents**.

---

## Step 6: Test the Agent

1. In Snowsight, navigate to **AI & ML → Cortex Agents**
2. Locate and open your newly created agent
3. Ask a question that maps to one of your defined metrics or models — for example:

> *"What was total revenue last month?"*

> *"Which customers placed the most orders in Q1?"*

> *"Show me order counts broken down by week."*

> **✅ Expected:** The agent returns a response grounded in the context dbt provides — metric definitions, lineage, and model documentation — all surfaced through the MCP Server. dbt generates the SQL, and the result is returned through the agent without any Snowflake Semantic View required.

---

## Troubleshooting

#### Connector shows "Not Connected" after authentication

Re-authenticate by navigating to **Settings → MCP Connectors** and clicking **Connect** again. OAuth tokens can expire after a period of inactivity.

#### API Integration creation fails
Ensure `OAUTH_ALLOWED_SCOPES = ('user_access', 'offline_access')` is included in the `API_USER_AUTHENTICATION` block. This is a known requirement — omitting it causes integration creation to fail regardless of other settings.

#### Agent returns no results or tool errors
Confirm your production environment has a recent successful job run that produced a `semantic_manifest.json`. The Semantic Layer cannot serve results without an up-to-date manifest.

#### API Integration creation fails with a hostname error
Snowflake requires hostnames to use hyphens (`-`) rather than underscores (`_`). If your dbt platform host URL contains underscores, contact your dbt account team to confirm the correct format for your account.

#### `execute_sql` tool returns an auth error
The `execute_sql` tool requires a Personal Access Token (PAT). Service tokens do not work for this tool. Update your integration to use a PAT if you need SQL execution capability.

#### OAuth errors or DCR not available
OAuth-based authentication for the remote dbt MCP Server may require an Enterprise or Enterprise+ dbt platform plan. If you encounter OAuth-related errors, confirm your plan tier with your dbt account team.

---

## Reference

| Resource | Link |
|---|---|
| dbt Semantic Layer docs | https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl |
| dbt remote MCP Server setup | https://docs.getdbt.com/docs/dbt-ai/setup-remote-mcp |
| dbt Semantic Layer + Snowflake quickstart | https://docs.getdbt.com/guides/sl-snowflake-qs |
| Snowflake MCP Connectors docs | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors |

### SQL reference: DCR-based API Integration and MCP Server

If you prefer to create these objects directly via SQL, use this pattern. Note the `OAUTH_ALLOWED_SCOPES` parameter — omitting it is a known cause of errors during integration creation.

```sql
-- Create the API integration using DCR
-- IMPORTANT: OAUTH_ALLOWED_SCOPES is required — omitting it will cause errors
CREATE API INTEGRATION dbt_mcp_api_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://YOUR_DBT_HOST_URL')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH_DYNAMIC_CLIENT
    OAUTH_RESOURCE_URL = 'https://YOUR_DBT_HOST_URL/api/ai/v1/mcp/'
    OAUTH_ALLOWED_SCOPES = ('user_access', 'offline_access')
  )
  ENABLED = TRUE;

-- Create the external MCP server
CREATE EXTERNAL MCP SERVER dbt_mcp_server
  WITH DISPLAY_NAME = 'dbt Semantic Layer'
  URL = 'https://YOUR_DBT_HOST_URL/api/ai/v1/mcp/'
  API_INTEGRATION = dbt_mcp_api_integration;
```

For custom OAuth (Option B from Step 3), the API Integration instead uses:

```sql
-- Create the API integration with custom OAuth credentials
-- IMPORTANT: OAUTH_ALLOWED_SCOPES is required — omitting it will cause errors
CREATE API INTEGRATION dbt_mcp_api_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://YOUR_DBT_HOST_URL')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH2
    OAUTH_CLIENT_ID = 'your_client_id'
    OAUTH_CLIENT_SECRET = 'your_client_secret'
    OAUTH_TOKEN_ENDPOINT = 'https://your_token_endpoint'
    OAUTH_AUTHORIZATION_ENDPOINT = 'https://your_auth_endpoint'
    OAUTH_CLIENT_AUTH_METHOD = CLIENT_SECRET_BASIC
    OAUTH_ALLOWED_SCOPES = ('user_access', 'offline_access')
  )
  ENABLED = TRUE;
```

---
