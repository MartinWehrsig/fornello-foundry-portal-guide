# Fornello Tool - Foundry Portal Setup

Connect the Fornello order API to an existing agent in the Microsoft Foundry portal.

## Required values

The API must already be deployed using [fornello-order-tool](https://github.com/MartinWehrsig/fornello-order-tool).
Obtain these values from the deployment owner:

| Value | Placeholder | Deployment source |
|---|---|---|
| API base URL | `https://<FUNCTION-APP>.azurewebsites.net/api` | `baseUrl` in `.local/connection.json` |
| OpenAPI URL | `https://<FUNCTION-APP>.azurewebsites.net/api/openapi.json` | `openapiUrl` in `.local/connection.json` |
| API key | `<FUNCTION-API-KEY>` | Value of `apiKey` in `.local/connection.json` |
| Header name | `x-functions-key` | `headerName` in `.local/connection.json` |

Replace placeholders with the deployment's actual values. Receive the API key
through a secure channel; do not put it in a prompt, schema or public repository.
Portal users do not need the deployment owner's entire connection file.

You need access to a Foundry project with an existing agent and model deployment,
plus permission to create or use project connections. Portal labels can vary.

## 1. Get the OpenAPI schema

Open the **OpenAPI URL** in a browser and copy the complete JSON document.
Alternatively, use the contents of the deployment-generated `.local/openapi.json`.

The document's `servers[0].url` must contain the actual API base URL.
Do not use the undeployed repository specification with `YOUR-FUNCTION-APP`.
The schema endpoint is public; no API key is required to retrieve it.

## 2. Open the tool dialog

In [Microsoft Foundry](https://ai.azure.com), select the project and existing
agent. Open **Tools** and add a **Custom > OpenAPI** tool.

In **Create an OpenAPI tool**, enter:

| Field | Value |
|---|---|
| **Name** | `fornello_orders` |
| **Description** | Use the text below. |
| **Authentication method** | **API key**, not Anonymous or Managed identity |
| **Connection** | Select the connection created in step 3. |
| **OpenAPI 3.0+ schema** | Paste the complete JSON from step 1, not its URL. |

Description:

```text
Submit confirmed demo pickup orders for Pizzeria Fornello. Validates menu items,
sizes, quantities and toppings, calculates prices and returns a simulated order
confirmation. No real orders are placed.
```

## 3. Create the API-key connection

Select **Create a new connection** in the tool dialog:

| Connection field | Value |
|---|---|
| **Connection name** | `fornello-order-api` |
| **Authentication type**, if shown | **API key** |
| **Target URL / Endpoint**, if shown | `https://<FUNCTION-APP>.azurewebsites.net/api` |
| Credential **Key / Name** | **`x-functions-key`** |
| Credential **Value / Secret** | **`<FUNCTION-API-KEY>`**, replaced with the actual key |

Store the credential as this key-value pair:

```text
Key:    x-functions-key
Value:  <FUNCTION-API-KEY>
```

**The credential name must be `x-functions-key`, not `apiKey`.**
`apiKey` is only the field name in the deployment's local JSON file.
Use its value as the secret. Do not rename the local JSON field.
`function_key` is the OpenAPI security scheme identifier, not the credential name.

Enter the secret without a `Bearer ` prefix, surrounding quotes or whitespace.
Create the connection, select it in the tool's **Connection** dropdown and click
**Create tool**. Save the agent changes.

## 4. Test

Start a new chat in the agent playground and send:

```text
Submit a demo pickup order: one large Margherita with extra mushrooms.
Use item_code P01, size large, quantity 1, extra_toppings ["mushrooms"],
and variant "". I confirm this exact demo order at EUR 12.40.
Call submit_order with confirmed=true.
```

Check the tool invocation, not just the generated chat response:

- Operation: `submit_order`.
- Result: `status: accepted_demo`, `demo: true`.
- Total: `total_eur: "12.40"`, `currency: "EUR"`.
- Order ID: starts with `DEMO-`.

For general menu requests, supply the agent with the same
[menu](https://github.com/MartinWehrsig/fornello-order-tool/blob/main/menu.txt).
Before ordinary submissions, obtain explicit confirmation of the exact order.
The API checks `confirmed=true` but cannot independently verify human consent.
Each successful call generates a new demo ID; avoid automatic retries.

## Troubleshooting

| Error | Check |
|---|---|
| `The required key: 'x-functions-key' is not present in the connection keys: apiKey` | The credential was named `apiKey`. Recreate or edit the connection using **`x-functions-key`** as the credential name, with the same secret value. Select the corrected connection and save. |
| HTTP `401 Unauthorized` | Select API-key authentication and the correct connection. Check the `x-functions-key` credential name and its actual value. If the owner rotated the Function key, update the connection secret. |
| HTTP `422` | The request reached the API, but the order failed validation. Read the returned error and correct the item, size, quantity, extras, variant or confirmation. |
| HTTP `404` or unresolved placeholder hostname | Use the deployment-generated schema. Its server URL must be `https://<ACTUAL-FUNCTION-APP>.azurewebsites.net/api`; the operation path is `/orders`. |
| Schema validation fails | Paste the full JSON document, not a URL or Markdown code fences. Preserve `operationId`, `security` and `components.securitySchemes`. |
| A chat reply appears, but no tool invocation | Confirm that the tool is attached to the selected agent and the changes are saved. Retry with the explicit test prompt above. |

## Reference

[Microsoft Learn: OpenAPI tools and authentication](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/openapi)
