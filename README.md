# Amazon listing image and video MCP server (Amazon Agent Iris) by Kuudo

Create Amazon listing images and product video from live catalog data, real product references, Amazon rules, and secure, human-approved workflows.

[![MCP](https://img.shields.io/badge/MCP-Model%20Context%20Protocol-8A2BE2?style=flat-square)](https://modelcontextprotocol.io/) [![Registry](https://img.shields.io/badge/MCP%20Registry-io.github.KuudoAI%2Famazon--agent--iris--mcp-blue?style=flat-square)](https://registry.modelcontextprotocol.io/v0.1/servers?search=io.github.KuudoAI%2Famazon-agent-iris-mcp&version=latest) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE) [![Community](https://img.shields.io/badge/Kuudo-community-D97757?style=flat-square)](https://github.com/KuudoAI/community)

Product page: https://www.kuudo.com/features/amazon-agent-iris/ · Docs: https://www.kuudo.com/docs/amazon-agent-iris/ · Pricing: https://www.kuudo.com/pricing.md

## Connect

Kuudo runs in your own cloud. The Community plan deploys one instance of each Amazon MCP server into your account, and your client connects to that deployment:

```json
{
  "mcpServers": {
    "amazon-agent-iris-mcp": {
      "url": "https://<your-host>/mcp",
      "headers": {
        "Authorization": "Bearer <your Kuudo API key>"
      }
    }
  }
}
```

Replace `<your-host>` with the hostname of your deployment and the bearer value with your Kuudo API key.

## What this repository is

This repository holds registry metadata and a catalog-only stub. Live execution runs in your Kuudo deployment. The server source is not published. The stub in `src/` answers `tools/list` with the catalog below, serves the same catalog as one resource (`kuudo://catalog/tools.json`), offers one prompt (`connect`) carrying the setup guidance, and returns an error with that guidance on any call, so registries and clients can inspect the surface without any access to Amazon.

### Inspect the catalog locally with Docker

The image runs the same catalog-only stub over stdio. It is not the live server.

```bash
docker build -t amazon-agent-iris-mcp .
docker run -i --rm amazon-agent-iris-mcp
```

Point a client at it with a stdio entry:

```json
{
  "mcpServers": {
    "amazon-agent-iris-mcp-catalog": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "amazon-agent-iris-mcp"]
    }
  }
}
```

## Tools

12 tools, read from a running instance of this server. Four of the twelve mirror the MCP resource and prompt methods, and two more are onboarding and server health. The remaining six generate images and video, hand back an upload URL for local source bytes, and track video jobs.

Each tool carries the argument schema the live server publishes, so a client can inspect the full call signature here before it connects to your deployment.

<details>
<summary>Every tool (12)</summary>

| Tool | Access | What it does |
| --- | --- | --- |
| `start_here` | read | Call this FIRST to understand how to use this image and video generation server. Returns workflow documentation for: - Available tools - HTTP image uploads - Editing photos - Long-running, resumable video generation - Generating new images - Downloading results |
| `generate_image` | write | Generate or edit images with **Google Gemini** (Nano Banana) from natural language instructions. For OpenAI gpt-image-2, use the sibling `generate_openai_image` tool. For **Amazon / Seller Central** listing images, FIRST read the `skill://amazon-product-image/SKILL.md` resource (via `read_resource`) for the marketplace image-compliance rules, then generate/audit against them. ## CHARACTER CONSISTENCY Use public image handles for character-consistent image generation: ``` generate_image( input_images=["agent-iris://images/img_12"], mode="generate", prompt="PRESERVE EXACTLY: character's facial features. NEW: Professional headshot...", model="gemini-3-pro-image" ) ``` Explicit generate mode uses the image handle as conditioning rather than an edit source. ## EDITING AN EXISTING IMAGE For local bytes, upload with POST /images first, then pass the returned handle. For a public hosted image, pass the HTTPS URL directly: ``` generate_image( input_images=["https://example.com/source.jpg"], mode="edit", prompt="Change the background to a beach sunset" ) ``` ## ITERATING ON A RESULT (high-fidelity chained edit) — PREFERRED Every generate/edit response returns an `interaction_id`. Pass it back to refine the SAME image while preserving the rest of the composition — no need to re-send the source. This beats re-editing from bytes for edit fidelity: ``` generate_image( interaction_id="v1_Chd...", # from the previous response prompt="Recolor the mug to beige. Change nothing else." ) ``` TTL: ~55 days (paid) / 1 day (free). ## COMPOSITING (Person from image A into scene B) Use multiple references with generate mode: ``` generate_image( input_images=["agent-iris://images/img_12", "gemini://files/beachRef"], mode="generate", prompt="Place character from image 1 into the beach scene from image 2..." ) ``` ## OPERATION PRESETS Use operation="icon", "pattern", "diagram", "storyboard", or "photo_repair" for strict defaults around model, aspect ratio, and prompt framing. ## Mode Auto-Detection: - interaction_id → EDIT mode - operation="photo_repair" → EDIT mode - operation="icon"/"pattern"/"diagram"/"storyboard" → GENERATE mode - operation="general" with one input image → EDIT mode - Multiple input images or pure prompt → GENERATE mode Returns both MCP image content blocks and structured JSON with metadata. |
| `generate_openai_image` | write | Generate images with **OpenAI gpt-image-2** via the Responses API. The additive OpenAI provider, sibling to `generate_image` (Google Gemini). Send your OpenAI key as `Authorization: Bearer <key>` — the tool name implies the provider. Iterate on a result with `previous_response_id` (OpenAI's analog of Gemini's `interaction_id`). |
| `create_upload_url` | write | Create the expiring signed URL needed to upload LOCAL image bytes. Use this when you have an image on disk (or raw bytes) to feed into generate_image: call this tool, then HTTP ``POST`` the bytes to the returned ``upload_url`` (no auth header needed — the URL's signed token authorizes it). The upload responds with a ``agent-iris://images/{id}`` handle to pass in generate_image's ``input_images``. Hosted http(s) image URLs do NOT need this — pass them to generate_image directly. |
| `server_status` | read | Return server diagnostics for auth, models, storage, uploads, resources, and health. |
| `generate_video` | write | Generate video with **Google Veo 3.1** from text, a first image, first/last frames, up to three reference images, or a Veo-generated video to extend. Inputs are bounded handles or http(s) URLs, never inline data; references, extensions, and 1080p/4k output require an 8-second request. Before planning or generating video, read the canonical `skill://amazon-video/SKILL.md` resource. It covers mode and model selection, prompt craft, recovery, and the additional compliance workflow for Amazon listing videos. Renders take 11 s to several minutes. Task-capable clients run this in the background; other clients receive `pending` and resume with `get_video_status(recovery_id)`. Reuse the same `recovery_id` after a timeout. If submission is reported as unknown, do not retry under a new id because Google may already have accepted the billed request. Results are signed download URLs and `agent-iris://videos/vid_N` handles. |
| `get_video_status` | read | Probe one video operation (exactly one of `recovery_id` / `operation_name`). Makes at most one Google status call and never waits for rendering. When the render is finished it downloads and stores the video in this call. Owner-scoped: only the key that submitted the operation can see it. |
| `list_video_operations` | read | List the video operations submitted with this key, newest first. Reads only the local operation store — no Google call. Use a listed `recovery_id` with `get_video_status` to resume or collect. |
| `list_resources` | read | List all available resources and resource templates. Returns JSON with resource metadata. Static resources have a 'uri' field, while templates have a 'uri_template' field with placeholders like {name}. |
| `read_resource` | read | Read a resource by its URI. For static resources, provide the exact URI. For templated resources, provide the URI with template parameters filled in. Returns the resource content as a string. Binary content is base64-encoded. |
| `list_prompts` | read | List all available prompts. Returns JSON with prompt metadata including name, description, and optional arguments. |
| `get_prompt` | read | Get a prompt by name with optional arguments. Returns the rendered prompt as JSON with a messages array. Arguments should be provided as a dict mapping argument names to values. |

</details>

## Set up your client
- [Quick Start: Activepieces MCP](https://www.kuudo.com/docs/quick-start/activepieces/)
- [Quick Start: Google Antigravity](https://www.kuudo.com/docs/quick-start/antigravity/)
- [Quick Start: ChatGPT Skills](https://www.kuudo.com/docs/quick-start/chatgpt-skills/)
- [Quick Start: ChatGPT](https://www.kuudo.com/docs/quick-start/chatgpt/)
- [Quick Start: Claude](https://www.kuudo.com/docs/quick-start/claude-ai/)
- [Quick Start: Claude Skills](https://www.kuudo.com/docs/quick-start/claude-app-skills/)
- [Quick Start: Claude Code MCP](https://www.kuudo.com/docs/quick-start/claude-code-mcp/)
- [Quick Start: Claude Cowork for Amazon Workflows](https://www.kuudo.com/docs/quick-start/claude-cowork-amazon/)
- [Quick Start: Claude Code Skills](https://www.kuudo.com/docs/quick-start/claude-skills/)
- [Quick Start: Codex Skills](https://www.kuudo.com/docs/quick-start/codex-skills/)
- [Quick Start: Codex](https://www.kuudo.com/docs/quick-start/codex/)
- [Quick Start: Hermes MCP](https://www.kuudo.com/docs/quick-start/hermes/)
- [Quick Start: Lovable](https://www.kuudo.com/docs/quick-start/lovable/)
- [Quick Start: n8n MCP](https://www.kuudo.com/docs/quick-start/n8n/)
- [Quick Start: NanoClaw MCP](https://www.kuudo.com/docs/quick-start/nanoclaw/)
- [Quick Start: OpenAI API](https://www.kuudo.com/docs/quick-start/openai-api/)
- [Quick Start: OpenClaw MCP](https://www.kuudo.com/docs/quick-start/openclaw/)
- [Quick Start: Perplexity MCP](https://www.kuudo.com/docs/quick-start/perplexity/)

## Playbooks

Operator guides grounded in Amazon's own documentation, each with the artifact the agent produces:
- [Diagnose a Suppressed Listing: Stranded to Buyable](https://www.kuudo.com/guides/seller-listing-suppression-diagnosis/)
- [Recover Buy Box Eligibility: Causes and Fixes](https://www.kuudo.com/guides/seller-buy-box-eligibility-recovery/)
- [Seeded Image Upgrades: Regenerating Listing Visuals From Your Current Photos, With Approval Gates](https://www.kuudo.com/guides/seller-listing-image-regeneration-seeded/)
- [Amazon FBM Orders Reports: What to Request and Why](https://www.kuudo.com/guides/seller-fbm-orders-reports/)
- [End-to-End Listing Optimization: From Live-Data Audit to a Previewed, Confirmed Patch](https://www.kuudo.com/guides/seller-listing-agentic-audit-to-patch/)

## How it compares
- [Kuudo vs Pixii for Amazon sellers](https://www.kuudo.com/compare/kuudo-vs-pixii/)

## Reads, writes, approvals

Read tools are safe to call freely. Write tools do work inside your deployment: they generate media, run a chained query, or hand back an upload URL. Each tool's access is recorded beside it in `tools.json`.

## Kuudo

- Website: https://www.kuudo.com/
- Docs: https://www.kuudo.com/docs/
- Guides: https://www.kuudo.com/guides/
- Community, bugs, and questions: https://github.com/KuudoAI/community
- Roadmap: https://github.com/orgs/KuudoAI/projects
- Machine-readable: https://www.kuudo.com/llms.txt · https://www.kuudo.com/pricing.md

Generated from KuudoAI/marketing. Do not edit by hand; changes are overwritten on the next sync.
