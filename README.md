# ReleasePad MCP Server

ReleasePad is a changelog and release notes tool for SaaS products. The ReleasePad MCP server lets AI assistants that speak the [Model Context Protocol](https://modelcontextprotocol.io/) (Claude, Claude Code, Cursor, VS Code, Windsurf and others) operate your changelog directly: list your products, search and read posts, draft and edit release notes, publish or schedule them, manage categories, upload images and read view analytics.

The server is hosted by ReleasePad and runs inside the ReleasePad application. There is nothing to install and no API key to copy: you add one URL to your client, log in to ReleasePad in the browser, and revoke access whenever you want.

This repository holds the public documentation for the server and the metadata used to list it in MCP directories (`server.json`). The server code itself is part of the ReleasePad application and is not published here.

## Who it is for

- Product and engineering teams that use ReleasePad and want release notes written where the work happens (a Claude conversation, a Claude Code or Cursor session, a CI agent).
- Anyone who wants to ask questions about what shipped, when, and who read it, without opening the ReleasePad dashboard.
- Developers building agents that need typed, structured access to a changelog.

A ReleasePad account on the Pro plan is required. See [Pricing](#pricing).

## Server facts

| | |
| --- | --- |
| Server URL | `https://pro.releasepad.io/mcp` |
| Transport | Streamable HTTP (MCP specification revision 2025-03-26 and later). JSON-RPC over `POST`; `GET` opens the optional server event stream; `DELETE` ends a session. There is no legacy HTTP+SSE endpoint. |
| Authentication | OAuth 2.1 authorization code flow with PKCE (`S256` only). Dynamic client registration (RFC 7591) is open, so clients register themselves. Discovery through RFC 8414 and RFC 9728 metadata. |
| Scopes | `read` (list and get tools) and `write` (create, update, delete and upload tools). A client that does not ask for scopes gets both. |
| Discovery URLs | `https://pro.releasepad.io/.well-known/oauth-authorization-server` and `https://pro.releasepad.io/.well-known/oauth-protected-resource` |
| Tools | 15, all typed with input and output schemas and MCP tool annotations. See [Tool reference](#tool-reference). |
| Plan | Pro plan (paid). Free accounts can complete the OAuth flow but every request answers `403` until the account upgrades. |
| Rate limits | ReleasePad does not publish a per token rate limit for the MCP endpoint. The endpoint sits behind Cloudflare, which may throttle abusive traffic. |
| Region and hosting | Hosted by ReleasePad (Fly.io behind Cloudflare). Requests must come from a client that can reach the public internet. |

## Connect

The server URL is the same for every client:

```
https://pro.releasepad.io/mcp
```

The first time a tool is used, a ReleasePad login page opens in your browser. You review what the client will be allowed to do, approve, and the client keeps a token. Every client you authorized is listed under **My Account > Connected Apps** (`https://pro.releasepad.io/account/settings/connected-apps`) with a **Revoke** button.

### Claude.ai, Claude Desktop and Claude mobile

Custom connectors are available on the Free, Pro, Max, Team and Enterprise plans (Free accounts are limited to one custom connector). A connector added on claude.ai is available in Claude Desktop and the mobile apps for the same account.

1. Open **Settings > Connectors** (shown as **Customize > Connectors** in the current interface).
2. Click **+** and choose **Add custom connector**.
3. Name it `ReleasePad` and paste `https://pro.releasepad.io/mcp` as the remote MCP server URL. Leave the OAuth client ID and secret fields in Advanced settings empty: ReleasePad registers the client automatically.
4. Click **Add**, then **Connect**. Log in to ReleasePad in the window that opens and click **Authorize**.

On Team and Enterprise plans an organization Owner adds the connector first under **Organization settings > Connectors > Add > Custom > Web**, and each member then clicks **Connect** under **Customize > Connectors**.

### Claude Code

```bash
claude mcp add --transport http releasepad https://pro.releasepad.io/mcp
```

Add `--scope user` to make the server available in every project instead of only the current one. Then, inside a Claude Code session, run `/mcp`, pick **releasepad** and follow the login link (recent versions also accept `claude mcp login releasepad` from the shell).

To share the configuration with your team, commit a `.mcp.json` file at the project root:

```json
{
  "mcpServers": {
    "releasepad": {
      "type": "http",
      "url": "https://pro.releasepad.io/mcp"
    }
  }
}
```

### Cursor

One click: open the [ReleasePad listing in the Cursor directory](https://cursor.directory/plugins/releasepad) and click **Add to Cursor**.

By hand: put this in `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project) and restart Cursor. Cursor prompts you to log in the first time a tool is used.

```json
{
  "mcpServers": {
    "releasepad": {
      "url": "https://pro.releasepad.io/mcp"
    }
  }
}
```

### VS Code and GitHub Copilot

Open the Command Palette, run **MCP: Add Server**, choose **HTTP**, enter `https://pro.releasepad.io/mcp` and the name `releasepad`. VS Code opens a browser window to authorize on the first connection.

Or create `.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "releasepad": {
      "type": "http",
      "url": "https://pro.releasepad.io/mcp"
    }
  }
}
```

### Windsurf

Open the MCP settings from the Cascade panel (**Windsurf Settings > Cascade > MCP Servers**) and add a custom server, or edit `~/.codeium/windsurf/mcp_config.json` directly:

```json
{
  "mcpServers": {
    "releasepad": {
      "serverUrl": "https://pro.releasepad.io/mcp"
    }
  }
}
```

Windsurf runs the OAuth login in the browser when the server is first used.

### Any MCP client

Any client that supports remote MCP servers over Streamable HTTP with OAuth 2.1 works the same way. The client needs to:

1. Send a request to `https://pro.releasepad.io/mcp` without a token. The answer is `401` with `WWW-Authenticate: Bearer resource_metadata="https://pro.releasepad.io/.well-known/oauth-protected-resource"`.
2. Read the protected resource metadata, then the authorization server metadata at `https://pro.releasepad.io/.well-known/oauth-authorization-server`.
3. Register itself with `POST https://pro.releasepad.io/oauth/register` (RFC 7591, no authentication). Redirect URIs are matched exactly, except that loopback URIs (`http://127.0.0.1` and `http://localhost`) may use any port. Custom schemes such as `cursor://` are accepted; `http://` on any other host is rejected.
4. Send the user to `https://pro.releasepad.io/oauth/authorize` with `response_type=code`, a `code_challenge` (`S256`) and optionally `scope=read` or `scope=read write`.
5. Exchange the code at `https://pro.releasepad.io/oauth/token` (form encoded or JSON body). Access tokens last one hour, refresh tokens 30 days and rotate on every use. Tokens can be revoked at `https://pro.releasepad.io/oauth/revoke`.
6. Call the server with `Authorization: Bearer <access token>`.

The [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) does all of this for you and is the quickest way to see every tool and call it by hand:

```bash
npx @modelcontextprotocol/inspector
```

Choose the **Streamable HTTP** transport, enter the server URL and click **Connect**.

Two more options for unusual setups:

- **API key instead of OAuth.** The endpoint also accepts a ReleasePad API key as the Bearer token (`Authorization: Bearer <api key>`), which is useful for scripts and for clients that support headers but not OAuth. API keys are not scoped: they get both `read` and `write`.
- **Clients that only speak stdio.** The community package [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) bridges a local stdio client to a remote OAuth server (`npx mcp-remote https://pro.releasepad.io/mcp`). It is not maintained by ReleasePad.

## Tool reference

Every tool is product scoped except `list_products`. Start with `list_products`: the numeric `id` of a product is the `product_id` every other tool takes. Posts and categories are also addressed by numeric ids (the `id` field in list results). Product slugs are not accepted.

Required inputs are marked with `*`. Access is the OAuth scope the tool needs.

| Tool | Title | Access | What it does | Inputs |
| --- | --- | --- | --- | --- |
| `list_products` | List products | read | Lists the products in your account, newest first, 10 per page. Each product includes its name, website, public changelog slug (`product_uri`), `posts_count` and logo. | `page` |
| `get_product` | Get product | read | Returns one product by its numeric id. | `product_id`* |
| `list_posts` | List posts | read | Lists a product's posts, newest first, 45 per page, with optional filters. Every post comes with its full Markdown body, so filter instead of paging through everything. | `product_id`*, `page`, `status` (`all`, `published`, `draft`, `to_be_published`), `title` (contains, case insensitive), `category_id` |
| `get_post` | Get post | read | Returns one post with its full Markdown body, blurb, category, draft flag, publication date and view count. | `product_id`*, `id`* |
| `create_post` | Create post | write | Creates a release note. Body is GitHub flavoured Markdown, at least 100 characters. Saved as a draft unless `draft=false`. A future `publish_at` schedules the post. | `product_id`*, `title`*, `body`*, `category_id`*, `draft`, `publish_at` (ISO 8601 or `YYYY-MM-DDTHH:MM` in the account timezone), `is_private`, `only_show_to_ids` |
| `update_post` | Update post | write | Partial update: only the fields sent change. A new `body` replaces the whole body. `draft=false` publishes, `draft=true` takes a post off the changelog. | `product_id`*, `id`*, `title`, `body`, `category_id`, `draft`, `publish_at`, `is_private`, `only_show_to_ids` |
| `delete_post` | Delete post | write | Deletes a post. It disappears from the changelog and cannot be restored through the server. To hide a post temporarily use `update_post` with `draft=true`. | `product_id`*, `id`* |
| `list_categories` | List categories | read | Lists a product's categories, sorted by name, no pagination. The `id` is the `category_id` posts need. | `product_id`* |
| `get_category` | Get category | read | Returns one category: name, badge color and product. | `product_id`*, `id`* |
| `create_category` | Create category | write | Creates a category. Categories show as a colored badge next to each post. | `product_id`*, `name`*, `background_color`* (hex, for example `#2ca2dd`) |
| `update_category` | Update category | write | Renames a category or changes its badge color. Partial input. | `product_id`*, `id`*, `name`, `background_color` |
| `delete_category` | Delete category | write | Deletes a category. Its posts are kept. Cannot be restored through the server. | `product_id`*, `id`* |
| `upload_image` | Upload image | write | Stores an image and returns its public URL to use in a post body as `![alt text](url)`. PNG, JPEG, GIF, WebP or SVG, 5 MB maximum, sent as base64 or as a `data:` URL. | `product_id`*, `image`* |
| `list_post_views` | List post views | read | Lists the individual views of one post (ip, country, city, timestamp), newest first, 10 per page. For the total only, use `view_counter` from `get_post`. | `product_id`*, `id`*, `page` |
| `list_changelog_views` | List changelog views | read | Lists the individual views of the product's public changelog page, newest first, 10 per page. | `product_id`*, `page` |

### Results and errors

- Every tool returns the same JSON the [ReleasePad management API](https://www.releasepad.io/api/) returns, both as text and as `structuredContent`, validated against the tool's output schema.
- Objects: `product` (`id`, `name`, `website`, `product_uri`, `public_token`, `posts_count`, `logo_url`, `created_at`, `updated_at`), `post` (`id`, `title`, `blurb`, `body`, `draft`, `is_private`, `only_show_to_ids`, `publish_at`, `view_counter`, `category`, `product_id`, `created_at`, `updated_at`), `category` (`id`, `name`, `background_color`, `product_id`, `created_at`, `updated_at`), `view` (`id`, `ip`, `user_metadata`, `country`, `city`, `created_at`) and `image` (`url`, `content_type`, `size`).
- List tools return a `pagination` object with `page`, `pages` and `count`. Call again with `page + 1` until `page` equals `pages`.
- `country` and `city` on views are filled in asynchronously and are `null` for recent views.
- Failures return `isError: true` with a JSend body: `{"status": "error", "message": "..."}` or `{"status": "fail", "data": {"field": ["problem"]}}`. Messages say what to fix, for example `No post with id 12 in product 5. Call list_posts with product_id=5 to see the valid ids.`

### Tool annotations

Every tool declares the four MCP annotations, so clients that honor them (Claude.ai and Claude Code among others) run reads without asking and confirm before destructive calls. `openWorldHint` is always `false`: the tools talk to ReleasePad only.

| Tools | `readOnlyHint` | `destructiveHint` | `idempotentHint` |
| --- | --- | --- | --- |
| `list_*`, `get_*` | true | false | true |
| `create_post`, `create_category`, `upload_image` | false | false | false |
| `update_post`, `update_category` | false | true | true |
| `delete_post`, `delete_category` | false | true | true |

## Example prompts

Product names and ids are examples; the assistant finds yours with `list_products`.

1. **Turn a pull request into a release note.** "Here is the PR description for the CSV export change: [paste]. Write a release note for the Reports product in ReleasePad. Match the tone of our last three published posts, file it under Improvement, and keep it as a draft."
2. **Ask what shipped.** "What did we release in August? Group by category and give me one line per post. Then list any drafts older than two weeks with their titles."
3. **Publish or schedule with one sentence.** "Publish the 'SSO for Google Workspace' draft." or "Schedule the 'New billing page' draft for next Tuesday at 9:00 our time."
4. **Clean up in bulk.** "Read every draft in the Mobile app product. Make them friendlier, fix any typos, and keep each one under 120 words. Leave them as drafts."
5. **Find out what people actually read.** "Which post got the most views in the last 30 days, and which countries were the readers in?"
6. **Reorganize categories.** "List the categories of the Reports product. Rename 'Bugfix' to 'Fix', give it the color #d9534f, and move every post whose title starts with 'Fix:' into it."

For coding agents, a rule like this in `CLAUDE.md` or your Cursor rules makes the agent that merged a change write it up while the context is fresh:

```markdown
After merging a user-facing change, draft a release note for it in
ReleasePad (product: Reports) as a draft and tell me the post id.
Never publish without asking.
```

## Guardrails and safety notes

- **Drafts by default.** A post created through the server is a draft unless the call explicitly passes `draft=false`. Drafts are not visible on the changelog page or in the in-app widget. Publishing is a separate, explicit step.
- **Publishing is public.** `draft=false` makes a post visible on your public changelog and widget immediately, or at `publish_at` if that is in the future. Review drafts before asking an assistant to publish.
- **Deletes cannot be undone from the server.** `delete_post` and `delete_category` are marked destructive and the server instructions tell assistants to confirm with you first, but nothing stops a client that ignores annotations. Prefer `update_post` with `draft=true` to hide a post.
- **Read only access.** A client that requests only the `read` scope gets a token that can list and read but not change anything; every write attempt is refused with a message asking to reconnect with write access. The consent page shows exactly the scopes the client asked for (it has no checkboxes of its own), and a client that asks for no scopes gets both `read` and `write`.
- **Account wide access.** A token acts with your permissions on every product of your account, not just one product.
- **Revocation.** **My Account > Connected Apps** lists every authorized client. Revoking ends its sessions immediately. Removing a user from the account also invalidates that user's tokens.
- **Consent page checks.** The consent page names the client, the redirect host you will be sent to and the URL the client claims to live at. Applications register themselves, so only approve requests you started from a tool you trust.
- **Token handling.** Access tokens expire after one hour, refresh tokens after 30 days and rotate on every use. Only hashes of codes and tokens are stored. A code or refresh token presented twice revokes the whole authorization.
- **Personal data in analytics.** `list_post_views` and `list_changelog_views` return viewer IP addresses and locations. Treat those results as personal data under your privacy obligations.
- **Validation limits.** Post bodies must be Markdown of at least 100 characters and at most 512 KB. Images are 5 MB maximum. Ids must come from the matching list tool.
- **Errors, not silence.** Wrong ids, missing scopes and invalid input come back as explicit `isError` results with a recovery hint, so assistants fix the call instead of guessing.

## Pricing

- **Free tier.** No credit card and no time limit, with a limited number of posts and one product. The MCP server is not included: a free account can complete the OAuth connection, but every request answers `403` ("The API is not available on trial accounts. Upgrade to a paid plan to use it.") until the account upgrades.
- **Pro: $35 per month per product.** Unlimited posts, unlimited team members, changelog analytics, GitHub integration, custom domain, API access and the MCP server. Pricing is per product, with no per seat fees.

Details and the current plan comparison are on the [pricing page](https://www.releasepad.io/pricing/). Start with a free account at [pro.releasepad.io/account/sign-up](https://pro.releasepad.io/account/sign-up).

## Documentation and support

- Feature page: [releasepad.io/features/mcp-server](https://www.releasepad.io/features/mcp-server/)
- Launch post with more prompts: [Your Changelog Just Got Hands: Meet the ReleasePad MCP Server](https://www.releasepad.io/blog/releasepad-mcp-server/)
- Management API (same operations over REST with an API key): [releasepad.io/api](https://www.releasepad.io/api/)
- Help center: [releasepad.io/help](https://www.releasepad.io/help/)
- Privacy policy: [releasepad.io/privacy-policy](https://www.releasepad.io/privacy-policy/)
- Terms: [releasepad.io/terms-and-conditions](https://www.releasepad.io/terms-and-conditions/)
- Support: [felix@releasepad.io](mailto:felix@releasepad.io)

Registry metadata for this server lives in [`server.json`](server.json); changes to this documentation are tracked in [`CHANGELOG.md`](CHANGELOG.md).

## License

The contents of this repository are released under the [MIT License](LICENSE). The ReleasePad service and the MCP server itself are governed by the [ReleasePad terms and conditions](https://www.releasepad.io/terms-and-conditions/).
