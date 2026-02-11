# Quick

[Quick](https://quick.shopify.io) is a tool at Shopify for quickly hosting static sites.
Sites are internal and behind google authentication. Only Shopify employees can see them.
Each site is hosted at [subdomain].quick.shopify.io

## Deployment

Files can be deployed by dragging and dropping into https://quick.shopify.io
They can also be deployed via the CLI

To install: `npm install -g @shopify/quick`

Deploy with `quick deploy dir your-subdomain`
This will upload the files in `dir` and be hosted at your-subdomain.quick.shopify.io
If a site already exists, a confirmation (y/n) will be presented to allow overwriting.
You should never attempt to bypass or automate confirmation. This is intended for users to confirm their actions.

When deploying, only deploy the output files. If you use a build system like vite, use /dist for example. This is for the final html & assets.

## Dev Environment

Sites can be previewed locally for rapid development iteration and testing using `quick serve`.
This will boot a small server in the current directory that will serve the site locally with full API functionality.

Important Notes:
- Quick sites are for frontend static assets only.
- Do not create anything that requires a backend.
- Each site must have an index.html file
- If you need additional functionality, use one of the APIs below.

## APIs

Quick exposes several APIs extend sites with serverless style functionality.

**Browser (UMD)**
Include with `<script src="/client/quick.js"></script>`

### Database (quick.db)

Firebase-like JSON database with real-time updates. Each site has its own isolated database.

```javascript
// Get collection
const posts = quick.db.collection("posts");

// Create
const doc = await posts.create({ title: "Hello", content: "World" });
// Returns: { id: "uuid", title: "Hello", content: "World", created_at: "...", updated_at: "..." }

// Read
const all = await posts.find();
const one = await posts.findById("id");

// Update
await posts.update("id", { title: "Updated" });

// Delete
await posts.delete("id");

// Query
const published = await posts
  .where({ status: "published" })
  .select(["title", "created_at"])
  .limit(10)
  .find();

// Real-time
const unsubscribe = posts.subscribe({
  onCreate: (doc) => console.log("New:", doc),
  onUpdate: (doc) => console.log("Updated:", doc),
  onDelete: (id) => console.log("Deleted:", id),
});
```

### AI (quick.ai), Agentic Workflows

Quick AI allows full utilization of OpenAI Client libraries in the browser via Shopify LLM proxy.

Include Quick Client Library:
```html
<script src="/client/quick.js"></script>
```

Initialize the client library:
```javascript
import OpenAI from 'https://cdn.jsdelivr.net/npm/openai/+esm';

const openaiClient = new OpenAI({
  baseURL: `/api/ai`,
  apiKey: 'not-needed', // Quick AI is pre-authenticated, this is a dummy value
  dangerouslyAllowBrowser: true
});
```

Use OpenAI Responses API to create and stream responses:
```javascript
// Stream a chat response
const messages = [
  { role: 'system', content: 'You are a helpful assistant' },
  { role: 'user', content: 'Explain async/await in JavaScript' }
];

const stream = await openaiClient.chat.completions.create({
  model: 'gpt-5.1',
  messages: messages,
  temperature: 1,
  stream: true,
});

let fullResponse = '';
for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) {
    fullResponse += content;
    // Update UI with each chunk
    document.getElementById('output').textContent = fullResponse;
  }
}
```

Use OpenAI Agents SDK for more advance use-cases such as Agentic Flows, MCP integration and more.

Basic Agent example
```javascript
import OpenAI from 'https://cdn.jsdelivr.net/npm/openai/+esm';
import { Agent, run, setDefaultOpenAIClient } from 'https://cdn.jsdelivr.net/npm/@openai/agents/+esm';

const openaiClient = new OpenAI({
  baseURL: `${window.location.origin}/api/ai`,
  apiKey: 'not-needed',
  dangerouslyAllowBrowser: true
});

// Configure Agent SDK
OpenAIAgents.setDefaultOpenAIClient(openaiClient);

// Create Agent
const agent = new OpenAIAgents.Agent({
  name: 'My Assistant',
  model: 'gpt-5.1',
  instructions: 'You are a helpful assistant'
});

// Run agent with streaming
const result = await OpenAIAgents.run(agent, 'Explain quantum computing', {
  stream: true
});

// Process streaming response
for await (const event of result) {
  if (event.type === 'raw_model_stream_event') {
    const content = event.data?.delta?.content || event.data?.delta;
    if (content) {
      document.getElementById('output').textContent += content;
    }
  }
}

await result.completed;
```

Use QuickMCPServerStreamableHttp.js to interoperate with MCP servers:

```html
<!-- Load QuickMCPServerStreamableHttp -->
<script src="https://quick.shopify.io/QuickMCPServerStreamableHttp.js"></script>

<script type="module">
import OpenAI from 'https://cdn.jsdelivr.net/npm/openai/+esm';
import { Agent, run, setDefaultOpenAIClient } from 'https://cdn.jsdelivr.net/npm/@openai/agents/+esm';

const openaiClient = new OpenAI({
  baseURL: `${window.location.origin}/api/ai`,
  apiKey: 'not-needed',
  dangerouslyAllowBrowser: true
});

setDefaultOpenAIClient(openaiClient);

// Initialize MCP server for tool access
const mcpServer = new QuickMCPServerStreamableHttp({
  url: `${window.location.origin}/api/ai/mcp/vault_set`,
  name: 'vault_set'
});
await mcpServer.connect();

// Create Agent with MCP tools
const agent = new Agent({
  name: 'My Assistant',
  model: 'gpt-5.1',
  instructions: 'You are a helpful assistant with database access',
  mcpServers: [mcpServer]
});

// Run agent with streaming
const result = await run(agent, 'What is Shopify Vacation Policy in Canada?', {
  stream: true
});

// Process streaming response with tool call logging
for await (const event of result) {
  if (event.type === 'raw_model_stream_event') {
    const content = event.data?.delta?.content || event.data?.delta;
    if (content) {
      document.getElementById('output').textContent += content;
    }
  }

  // Log when tools are used
  if (event.type === 'run_item_stream_event' && event.name === 'tool_call_started') {
    console.log(`🔧 Tool: ${event.item.name}`);
  }
}

await result.completed;
```

### File Storage (quick.fs)

File upload and management with progress tracking.

```javascript
// Upload file
const result = await quick.fs.uploadFile(file, {
  onProgress: (progress) => console.log(`${progress.percentage}%`),
});
// Returns: { files: [{ url, fullUrl, size, mimeType }] }

// Multiple files
const results = await quick.fs.upload(files);

// Naming strategies: 'uuid', 'timestamp', 'hybrid' (default), 'original'
await quick.fs.upload(files, { strategy: "original" });
```

### WebSocket (quick.socket)

A simple wrapper around Socket.IO for building real-time multiplayer experiences. It handles room management, user presence, and state synchronization automatically.

Rooms are automatically isolated by subdomain: joining a room like "lobby" on my-app.quick.shopify.io will only connect you with users on the same my-app subdomain.

Do not add realtime functionality unless asked to.

```javascript
const room = quick.socket.room("lobby");

// Set up handlers before calling join() to avoid missing any messages

// Presence/state events (per-user)
room.on("user:join", (user) => ...);
room.on("user:leave", (user) => ...);
room.on("user:state", (prev, next, user) => ...);

// Connection lifecycle
room.on("connect", () => console.log("room connected"));
room.on("disconnect", (reason) => console.log("room disconnected", reason));

await room.join();

// List all users (as a Map where the key is the socketId: Map<socketId, User>)// This variable is kept up to date as users join and leave
room.users

// Each user object always includes: socketId, name, and email.
// Other fields (slackHandle, slackId, slackImageUrl, title) may be present if available.
// Every user also has a `state` property that stays in sync as it changes.

// Get current user state
room.user.state

// Update your own state
// Note: this does a partial merge on server and only updates the keys you pass in.
room.updateUserState({ cursorPos: [102,560] });

// Send ephemeral custom events (not part of state)
room.emit("ping", { t: Date.now() });

// Listen to ephemeral custom events
room.on("ping", (data, user) => console.log("ping", data, user));

// Leave room
await room.leave();
```

```javascript
// Get user info
const user = await quick.id.waitForUser();
// Returns: { email, firstName, fullName, timestamp, slackImageUrl, slackHandle, slackId, title, github }

// Direct access
console.log(quick.id.email); // "john@shopify.com"
console.log(quick.id.fullName); // "John Doe"
```

### Site Management (quick.site)

Create, manage, and delete Quick sites programmatically.

```javascript
// Create a site from files
const htmlContent = `<!DOCTYPE html>
<html><body><h1>Hello World!</h1></body></html>`;
const indexFile = new File([new Blob([htmlContent])], "index.html");

const result = await quick.site.create("my-site-name", [indexFile]);
// Returns: { message: "Site created successfully", url: "https://my-site-name.quick.shopify.io" }

// Force overwrite existing site
await quick.site.create("my-site-name", files, { force: true });

// Get site information
const siteInfo = await quick.site.get("my-site-name");
// Returns: { subdomain: "my-site-name", url: "...", lastModified: "...", modified-by: "..." }
// Returns: null if site doesn't exist

// Delete a site (with a popup confirmation prompt)
await quick.site.delete("my-site-name");

// Delete without confirmation prompt
await quick.site.delete("my-site-name", { confirm: true });
```

### Data warehouse (quick.dw)

Query data from BigQuery. This uses the user's permissions and the user needs to go through the oauth flow first.

```javascript
// Request permissions first
const authResult = await quick.auth.requestScopes([
  "https://www.googleapis.com/auth/bigquery",
]);
// If authResult.hasRequiredScopes is true, permissions were granted
// If this is the first time they request it, a popup will appear asking the person
// to grant permission. This will go through oauth flow and redirect back to the page.

// Simple query
const results = await quick.dw.querySync(
  "SELECT * FROM dataset.table LIMIT 10"
);
// Returns: { results: [...], rowCount: 123 }

// Query with parameters (@ prefix for BigQuery named parameters)
const data = await quick.dw.querySync(
  "SELECT * FROM dataset.users WHERE age > @age",
  { name: "age", value: 21 }
);

// Query with options
const results = await quick.dw.querySync(sql, params, {
  timeoutMs: 60000, // Default: 30000
  maxResults: 1000, // Limit rows returned
});
```

### Slack (quick.slack)

Send messages to Slack channels.

```javascript
// Simple message (user slack id or channel id)
await quick.slack.sendMessage("#W02AJD7R7LQ", "Hello team!");

// Rich message with blocks
await quick.slack.sendMessage("#W02AJD7R7LQ", "Deploy complete", {
  blocks: [
    {
      type: "section",
      text: { type: "mrkdwn", text: "*Deploy complete* ✅" },
    },
  ],
});

// Thread reply
const msg = await quick.slack.sendMessage("W02AJD7R7LQ", "Original message");
await quick.slack.sendMessage("W02AJD7R7LQ", "Reply", {
  thread_ts: msg.slack_response.ts,
});

// Alerts (info/warning/error/success)
await quick.slack.sendAlert("W02AJD7R7LQ", "Database down", "error");

// Status updates (online/offline/maintenance/degraded)
await quick.slack.sendStatus(
  "W02AJD7R7LQ",
  "online",
  "All systems operational"
);

// Code snippets
await quick.slack.sendCode(
  "#dev",
  'console.log("Hello");',
  "javascript",
  "Example"
);

// Tables
await quick.slack.sendTable(
  "#reports",
  "Sales Report",
  ["Product", "Sales"],
  [
    ["Widget", 100],
    ["Gadget", 200],
  ]
);
```
