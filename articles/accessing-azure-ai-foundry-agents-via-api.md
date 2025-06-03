# Accessing Azure AI Foundry Agents via API

Azure AI Foundry agents live under a **project** scope, which means you cannot call them the same way you invoke Azure OpenAI “assistants.” Instead, your API calls must target the Foundry project endpoint—not the generic OpenAI-style URL. Below is a step-by-step, humanized guide (complete with code snippets and examples) that shows how to authenticate, list your agents, and run them successfully. By following this walkthrough, you’ll avoid the dreaded “agent not found” response and get your agents running in no time.

---

## Overview

1. **Why the “generic OpenAI” endpoint won’t work**
   Azure AI Foundry agents are nested under a project in your Foundry resource. Any call that omits the `/api/projects/<project-name>/…` prefix ends up in the wrong scope, and you’ll see:

   ```
   {
     "error": {
       "code": "NotFound",
       "message": "agent not found"
     }
   }
   ```

   ([learn.microsoft.com][1])

2. **What you need**

   * An **Azure AI Foundry** resource with at least one **project**.
   * The **Project endpoint** URL (found on the project’s Overview page).
   * A valid **Azure AD token** (or a resource key for quick tests).
   * Proper **RBAC permissions** (e.g., “Azure AI Account Owner” or a project-level role).
   * (Optional) The **Azure AI Projects** Python SDK (`azure-ai-projects`), which simplifies REST calls.

3. **Key steps**

   1. Locate your **project endpoint**.
   2. Authenticate via **Azure AD** (or use a resource key).
   3. **List** agents to get their `agentId`.
   4. **Run** an agent via the `/run` endpoint.
   5. (Optional) Use the **Python SDK** for an even smoother experience.

---

## 1. Find Your Project Endpoint

Before you write any code, grab the correct endpoint URL:

1. Go to the Azure Portal and navigate to your **Azure AI Foundry** resource.
2. On the left side, click **Projects** and then select the project you want.
3. Under the **Overview** tab, copy the **Project endpoint**. It looks like this:

   ```
   https://<your-ai-service>.services.ai.azure.com/api/projects/<project-name>
   ```

   ([learn.microsoft.com][2])

This endpoint is your “home base” for all agent-related REST calls. All subsequent URLs will be tacked onto this root.

---

## 2. Authenticate with Azure AD (or Use a Resource Key)

### 2.1. Azure AD (Recommended)

For production scenarios, you should obtain an Azure AD token. In most SDKs or CLI contexts, `DefaultAzureCredential()` does this for you automatically if you’ve already run `az login` or set up a managed identity.

* **CLI method** (PowerShell or Bash):

  ```bash
  az account get-access-token --resource https://ml.azure.com/ --query accessToken --output tsv
  ```

  Copy the returned token and use it in the `Authorization: Bearer <token>` header of your HTTP requests.

* **SDK method** (Python example):

  ```python
  from azure.identity import DefaultAzureCredential

  credential = DefaultAzureCredential()
  ```

  You’ll pass `credential` when creating your Foundry client in code. ([learn.microsoft.com][2])

### 2.2. Resource Key (Quick Testing Only)

If you’re experimenting and want a simpler route, generate a **resource key** under **Keys and Endpoint** in your Foundry resource. Then send it as:

```
Authorization: Bearer <YOUR_RESOURCE_KEY>
```

However, note that resource keys are **not** recommended for long-term or multi-user scenarios; they lack the fine-grained security of Azure AD.

---

## 3. List Your Agents (Get the `agentId`)

You cannot run an agent until you know its `agentId`. To retrieve that, call the List Agents endpoint under your project:

```http
GET https://<your-ai-service>.services.ai.azure.com/api/projects/<project-name>/agents?api-version=2025-05-16
Authorization: Bearer <YOUR_AAD_TOKEN_OR_KEY>
```

A successful response will look something like this:

```json
[
  {
    "id": "f9e8a7c4-1b2d-4a55-aa93-f9b2e44e5cd1",
    "description": "MyFirstAgent",
    "createdDateTime": "2025-05-01T14:22:33Z",
    "provisioningState": "Succeeded"
    // … other metadata …
  },
  {
    "id": "a1b2c3d4-0123-4567-89ab-cdef01234567",
    "description": "OrderProcessingAgent",
    "createdDateTime": "2025-05-14T09:11:02Z",
    "provisioningState": "Succeeded"
    // … other metadata …
  }
]
```

Copy the `id` field for the agent you want to run.

> **Pro tip:** If you have many agents, you can filter by description:
>
> ```
> GET …/agents?api-version=2025-05-16&$filter=description eq 'OrderProcessingAgent'
> ```
>
> That way, you only get back the agent(s) whose description matches “OrderProcessingAgent.”

---

## 4. Run Your Agent (The `/run` Call)

Now that you have the `agentId` and a valid bearer token, you can invoke the agent:

```http
POST https://<your-ai-service>.services.ai.azure.com/api/projects/<project-name>/agents/<agentId>:run?api-version=2025-05-16
Content-Type: application/json
Authorization: Bearer <YOUR_AAD_TOKEN_OR_KEY>

{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user",   "content": "Hello, Foundry agent!"   }
  ]
}
```

* **URL breakdown**

  * `https://<your-ai-service>.services.ai.azure.com` → your Foundry resource’s Fully Qualified Domain Name (FQDN).
  * `/api/projects/<project-name>/agents/<agentId>:run` → “run this specific agent under the given project.”
  * `?api-version=2025-05-16` → ensures you’re using the correct REST API version.

* **Request body**
  A standard chat payload: an array of messages (`role` + `content`).

If everything (token, `agentId`, RBAC, and agent provisioning) is correct, you’ll receive a JSON response filled with the agent’s reply:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hey there! How can I help you today?"
      }
    }
  ],
  "model": "gpt-4o",
  "usage": {
    "promptTokens": 12,
    "completionTokens": 11,
    "totalTokens": 23
  }
}
```

Congratulations—you’ve just run your Foundry agent!

---

## 5. Python SDK Example (Quickstart)

If you prefer using a higher-level SDK rather than raw HTTP calls, the **Azure AI Projects** Python library (`azure-ai-projects`) wraps all of the above for you. Below is a minimal script that lists agents and runs one:

```python
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

# 1) Set your environment variable to the project endpoint:
#    e.g. PROJECT_ENDPOINT="https://<your-ai-service>.services.ai.azure.com/api/projects/<project-name>"
project_endpoint = os.getenv("PROJECT_ENDPOINT")

# 2) Authenticate using DefaultAzureCredential
credential = DefaultAzureCredential()

# 3) Create the project client
project_client = AIProjectClient(
    endpoint=project_endpoint,
    credential=credential
)

# 4) List agents to find your agentId
agents = project_client.agents.list_agents()
print("Available agents:")
for agent in agents:
    print(f"  • id={agent.id}, description={agent.description}")

# Assume we pick the first agent in the list
agent_id = agents[0].id

# 5) Run the agent with a simple conversation
response = project_client.agents.run_agent(
    agent_id=agent_id,
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Tell me a joke."}
    ]
)

print("Agent response:", response.choices[0].message.content)
```

* **Install the library**:

  ```bash
  pip install azure-ai-projects azure-ai-identity
  ```

  This automatically pulls in `azure-ai-agents` under the hood. ([pypi.org][3])

* **How it works**

  1. `AIProjectClient(endpoint, credential)` points to your project endpoint.
  2. `project_client.agents.list_agents()` executes the REST call to `GET /agents`.
  3. `project_client.agents.run_agent(...)` maps to `POST /agents/{agentId}:run`.

Using the SDK takes care of token acquisition, RBAC, and URL composition for you.

---

## 6. Common Pitfalls & Troubleshooting

Even with the right URL and a valid token, you might still hit snags. Here are the usual culprits:

1. **“Agent not found”**

   * If your request URL is missing `/api/projects/<project-name>/…`, the service interprets it as an attempt to invoke an Azure OpenAI assistant rather than a Foundry agent. Always include `…/api/projects/<project-name>/agents/`. ([learn.microsoft.com][1])

2. **Invalid or expired token**

   * An expired or malformed token yields `401 Unauthorized`. Re-run:

     ```bash
     az account get-access-token --resource https://ml.azure.com/ --query accessToken --output tsv
     ```

     Or recreate your `DefaultAzureCredential()` instance.

3. **RBAC missing or insufficient permissions**

   * The identity (user or service principal) calling the API needs either:

     * **Azure AI Account Owner** or **Contributor** at subscription/resource scope, **or**
     * A **project-level role** assigned via **Access Control (IAM)** on the Foundry project.
   * Without this, you’ll see `403 Forbidden` when listing or running agents.

4. **Incorrect `api-version` or URL path**

   * Using the wrong version (e.g., `2024-12-01-preview`) can cause `404 Not Found`. Stick to `2025-05-16` (or the latest documented).

5. **Project endpoint vs. hub endpoint**

   * If you accidentally use a **Hub** endpoint (e.g., `https://<resource>.services.ai.azure.com/api/hubs/<hub-name>`), you won’t reach your project’s agents. Confirm you’re targeting `/api/projects/<project-name>`.

6. **Agent extension provisioning failure**

   * Before running, Foundry agents must be provisioned in your project. If the agent’s provisioningState is not `Succeeded`, it won’t run. Check the portal and wait for provisioning to finish. ([learn.microsoft.com][4])

---

## 7. Tips for a Smoother Experience

* **Use environment variables** for sensitive values (e.g., `PROJECT_ENDPOINT`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`). Don’t hard-code tokens or endpoints in source control.
* **Enable telemetry/tracing** if using the SDK: call `project_client.enable_telemetry()` to send logs to Application Insights. ([learn.microsoft.com][5])
* **Test with curl/Postman** first to nail down your URL, headers, and body format before writing code. It’s far easier to debug a single HTTP call than a full application.
* **Pin your SDK version** (e.g., `azure-ai-projects==1.0.0b11`) since Foundry libraries are still evolving in preview. Locking to a known version avoids unexpected breaking changes. ([pypi.org][3])

---

## 8. Summary

* **Azure AI Foundry agents live under a project**—they are not invocable via the generic OpenAI “assistants” endpoint.
* **Always start with your project endpoint** (`https://<your-ai-service>.services.ai.azure.com/api/projects/<project-name>`).
* **Authenticate via Azure AD** (or a resource key for quick tests), then **list your agents** to obtain the `agentId`.
* **Call** `POST /agents/{agentId}:run` with a valid JSON `messages` array to get the agent’s response.
* **Check your RBAC**—the caller needs appropriate roles, or you’ll see `403 Forbidden`.
* **Verify `api-version`** and your **URL path** carefully; even one typo sends you to a dead end.

Follow these steps, and you’ll be running your Foundry agents from command-line scripts, GitHub workflows, or production applications—without ever facing “agent not found” again. Happy coding!

---

## References

1. Azure AI Foundry Agent Service documentation overview: ([learn.microsoft.com][4])
2. Quickstart: Create a new Azure AI Foundry Agent Service project: ([learn.microsoft.com][6])
3. Azure AI Foundry SDK client libraries (Python): ([learn.microsoft.com][2])
4. Azure AI Foundry Portal Overview (finding the project endpoint): ([ai.azure.com][7])
5. What is Azure AI Foundry Agent Service? (concepts and architecture): ([learn.microsoft.com][5])
6. Listing agents via REST API:
7. Getting Azure AD access token (CLI):
8. Differences between Foundry agents and OpenAI assistants: ([learn.microsoft.com][1])
9. AI Projects client library for Python (installation & setup): ([pypi.org][3])
10. Azure AI Foundry role-based access control guidance:

[1]: https://learn.microsoft.com/en-us/azure/ai-foundry/?utm_source=chatgpt.com "Azure AI Foundry documentation - Learn Microsoft"
[2]: https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/sdk-overview?utm_source=chatgpt.com "Azure AI Foundry SDK client libraries - Learn Microsoft"
[3]: https://pypi.org/project/azure-ai-projects/?utm_source=chatgpt.com "Azure AI Projects client library for Python - PyPI"
[4]: https://learn.microsoft.com/en-us/azure/ai-services/agents/?utm_source=chatgpt.com "Azure AI Foundry Agent Service documentation - Learn Microsoft"
[5]: https://learn.microsoft.com/en-us/azure/ai-services/agents/overview?utm_source=chatgpt.com "What is Azure AI Foundry Agent Service? - Learn Microsoft"
[6]: https://learn.microsoft.com/en-us/azure/ai-services/agents/quickstart?utm_source=chatgpt.com "Quickstart - Create a new Azure AI Foundry Agent Service project"
[7]: https://ai.azure.com/?utm_source=chatgpt.com "Azure AI Foundry"
