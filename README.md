# n8n-nodes-better-http-request

An enhanced **HTTP Request** community node for [n8n](https://n8n.io) that adds automatic retry logic, per-item fallback responses, an inline post-processing code block, and a configurable node label — all on top of every capability already provided by the built-in HTTP Request node.

## Features

- **All standard HTTP Request capabilities** – HTTP methods (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS), query parameters, request headers, JSON/form/raw bodies, file uploads, pagination, batching, authentication, SSL certificates, and more.
- **Retry only failed items** – automatically re-sends requests that failed with configurable HTTP status codes. Unlike the built-in node's retry which re-runs every item, this feature retries *only* the items that failed, leaving successful items untouched.
  - Configurable maximum retry attempts (default: 3).
  - Configurable delay between retries (default: 1 000 ms).
  - Configurable list of retriable status codes (default: `429,500,502,503,504`).
  - Respects the `Retry-After` response header for HTTP 429 responses.
- **Fallback response on failure** – instead of emitting an `{ error }` item when a request fails, emit a fully configurable synthetic response shaped like `{ statusCode, headers, body }`. All fields support n8n expressions.
- **Post-processing code block** – run JavaScript directly inside the node after every request completes. Transform output, log to the execution log, publish side-effects — no extra Code node required.
- **Node label** – stamp each node instance with an emoji or short text label that appears on the canvas subtitle (e.g. `🚀 GET: https://api.example.com`).

## Installation

### In your n8n instance

1. Open **Settings → Community Nodes**.
2. Click **Install a community node**.
3. Enter `n8n-nodes-better-http-request` and confirm the installation.

### Manual install on self-hosted

```bash
npm install n8n-nodes-better-http-request
```

Then restart n8n.

## Usage

After installation the node appears in the node palette as **Better HTTP Request**.

### Basic request

1. Add the **Better HTTP Request** node to your workflow.
2. Set the **Method** (GET, POST, …) and the **URL**.
3. Optionally configure **Query Parameters**, **Headers**, and a **Body** in the respective sections.
4. Execute the workflow.

### Retry failed items

1. Enable **Continue On Fail** on the node (gear icon → *Continue On Fail*).
2. Open the **Options** section and turn on **Retry Failed Items**.
3. Adjust **Max Retries**, **Retry Delay (ms)**, and **Retry On Status Codes** to your needs.

When the workflow runs, any item whose request returns one of the configured status codes will be retried up to the specified number of times. All other items pass through immediately without waiting.

### Fallback response on failure

The fallback feature lets you replace the `{ error }` output item with a synthetic success-shaped response when a request fails — useful when downstream nodes expect consistent data regardless of HTTP errors.

1. Enable **Continue On Fail** on the node.
2. Open **Options** → turn on **Use Fallback Response**.
3. Set **Fallback Response Body** to any JSON value (expressions supported, e.g. `={{ { "status": "unavailable" } }}`).
4. Optionally add **Fallback Response Headers** (key-value pairs, expressions supported).
5. Set **Fallback Status Code** (default: `200`).

When a request fails, the output item will look like a normal full HTTP response:

```json
{
  "statusCode": 200,
  "headers": { "content-type": "application/json" },
  "body": { "status": "unavailable" }
}
```

This makes `$json.body`, `$json.statusCode`, and `$json.headers` available to downstream nodes — the same keys produced by **Return Full Response** mode on a successful request.

### Post-processing code block

Run JavaScript after every request completes without adding a separate Code node. The code fires on the **final, fully-resolved output** (after retries and fallback substitution).

1. Enable the **Post-Processing Code** toggle in the main panel.
2. Write JavaScript in the embedded code editor (full syntax highlighting).
3. Optionally `return items` (or a new array) to replace the output.

#### Sandbox API

| Variable | Description |
|---|---|
| `items` | Array of output items (`{ json, binary?, pairedItem }`). Mutate in place or return a new array. |
| `$input.all()` | Original items received by this node (before the HTTP request). |
| `$input.first()` | Shorthand for the first input item. |
| `$node` | Read-only node metadata: `{ name, id, type }`. |
| `console` | `log`, `info`, `warn`, `error`, `debug` — all write to the n8n execution log tagged `[postProcess]`. |

`async`/`await` is fully supported.

#### Examples

```js
// Add a field to every output item
for (const item of items) {
  item.json.processedAt = new Date().toISOString();
}
return items;
```

```js
// Log and pass through unchanged (no return needed)
console.log($node.name, 'returned', items.length, 'items');
```

```js
// Filter to only successful items
return items.filter(item => !item.json.error);
```

```js
// Async side-effect (e.g. notify a webhook) then pass through
await fetch('https://hooks.example.com/notify', {
  method: 'POST',
  body: JSON.stringify({ count: items.length }),
});
return items;
```

> **Error handling** — if the code throws and *Continue On Fail* is off the node halts. If *Continue On Fail* is on, an error item is appended and execution continues.

### Node label

Add an emoji or short text in the **Node Label** field at the top of the panel. It appears before the method and URL in the node canvas subtitle, making complex workflows easier to scan at a glance.

```
🚀  GET: https://api.example.com/users
✅  POST: https://crm.example.com/leads
🔴  DELETE: https://api.example.com/session
```

## Options reference

| Option | Default | Description |
|---|---|---|
| **Retry Failed Items** | `false` | Enable automatic retry for failed items. Requires *Continue On Fail*. |
| **Max Retries** | `3` | Maximum retry attempts per failed item (1–10). |
| **Retry Delay (ms)** | `1000` | Milliseconds between retries. For HTTP 429, `Retry-After` takes precedence. |
| **Retry On Status Codes** | `429,500,502,503,504` | Comma-separated status codes that trigger a retry. |
| **Use Fallback Response** | `false` | Emit a synthetic `{ statusCode, headers, body }` item on failure instead of `{ error }`. Requires *Continue On Fail*. |
| **Fallback Response Body** | `{}` | JSON body of the fallback item. Supports expressions. |
| **Fallback Response Headers** | *(empty)* | Key-value headers to include in the fallback item. Supports expressions. |
| **Fallback Status Code** | `200` | Status code to include in the fallback item (100–599). |
| **Batching** | – | Split items into batches with a configurable delay between them. |
| **Timeout** | `10000` | Time in ms to wait for the server to start responding before aborting. |
| **Send Credentials on Cross-Origin Redirect** | `false` | Forward auth headers when following cross-origin redirects. |

## Development

```bash
# Install dependencies
npm ci

# Build
npm run build

# Run tests
npm test
```

The project is written in TypeScript. The compiled output is placed in `dist/`.

## License

MIT
