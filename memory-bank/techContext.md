# Technical Context

This document outlines the technologies used, development setup, technical constraints, dependencies, and configuration for the Fauxmium project. It reflects the current codebase.

## Technologies

- Node.js — runtime environment
- Puppeteer — controls a visible Chrome instance, installs/sets up the extension, and intercepts network requests
- Vercel AI SDK (`ai`) — unified streaming interface for multiple text providers
- AI provider SDKs:
  - `@openrouter/ai-sdk-provider` (text, images, and videos via Vercel AI SDK; default provider)
  - `@ai-sdk/google` (text, images, and videos via Vercel AI SDK)
  - `@ai-sdk/openai` (text via Vercel AI SDK)
  - `@ai-sdk/anthropic` (text via Vercel AI SDK)
  - `@ai-sdk/groq` (text via Vercel AI SDK)
- `yargs` — command-line interface and subcommands
- `dotenv` — environment variable configuration

## Providers and Models

- Text providers: OpenRouter, Google (Gemini), OpenAI, Anthropic, Groq (via Vercel AI SDK)
- Image providers: OpenRouter, Google (Gemini)
- Video providers: OpenRouter, Google (Gemini)
- Defaults and examples:
  - Default text provider: OpenRouter with model `google/gemini-3.5-flash`
  - Default image provider: OpenRouter with model `google/gemini-2.5-flash-image`
  - Default video provider: OpenRouter with model `google/veo-3.1-fast`
- Notes:
  - OpenRouter accepts free-form model slugs (no `choices` restriction).
  - OpenRouter requests `usage: { include: true }` and reports the exact per-request USD cost in `providerMetadata.openrouter` (chat: `usage.cost`; image/video: `cost`), which the cost calculator uses directly instead of Helicone pricing.
  - Selecting an unsupported image/video provider will cause the pipeline to error; the server returns a transparent placeholder image.

## Environment Variables

Set in a `.env` file at the project root or via your shell environment:

- OPENROUTER_API_KEY — for OpenRouter text, images, and videos (default provider)
- GEMINI_API_KEY or GOOGLE_API_KEY — for Google text and images
- OPENAI_API_KEY — for OpenAI text
- ANTHROPIC_API_KEY — for Anthropic text
- GROQ_API_KEY — for Groq text

index.js resolves keys automatically per selected provider. You can also pass explicit keys via CLI flags.

## Development Setup

1. Install dependencies:
   - npm install
2. Configure API keys:
   - Create `.env` and add the relevant keys (see Environment Variables).
3. Run:
   - npx fauxmium
   - Optional flags:
     - --hostname|-H (default 127.0.0.1)
     - --port|-p (default 3001)
     - --devtools (open DevTools on launch)
     - --model|-m (text model)
     - --image-provider (openrouter or gemini/google)
     - --image-model|-i (image model)
     - --api-key (text provider override)
     - --image-api-key (image provider override)

## CLI Usage Overview

- Default command (no subcommand):
  - Text: OpenRouter (`google/gemini-3.5-flash`), Images: OpenRouter (`google/gemini-2.5-flash-image`), Videos: OpenRouter (`google/veo-3.1-fast`)
- Provider commands for text:
  - fauxmium openrouter
  - fauxmium gemini
  - fauxmium openai
  - fauxmium anthropic
  - fauxmium groq
- Image configuration:
  - Per provider, there is a nested `images` subcommand that adjusts image settings; supported image providers are openrouter and gemini/google.
  - Example: fauxmium gemini images --image-model gemini-2.5-flash-image

## Server Endpoints

- GET /html — streams AI-generated HTML (inside ```html fenced blocks), extracted and sent to the browser
- GET /image — returns AI-generated image binary (PNG/JPEG etc.) or a 1x1 transparent PNG on failure
- GET /cost — returns in-memory session usage/cost summary

## Streaming and Processing Pipeline

- server/processHTML.js:
  - Builds prompts from prompts/html.txt and request metadata
  - Streams text via lib/aiAdapter.js (Vercel AI SDK)
  - Pipes through lib/processChunks.js with:
    - lib/costCalculator.js — accumulates per-request/session cost from usage metadata
    - lib/streamCodeBlocks.js — incrementally extracts ```html code fences
- server/processImage.js:
  - Builds prompts from prompts/image.txt
  - Uses lib/aiAdapter.js generateImage for OpenRouter and Google/Gemini
  - Sends binary image response (fallback to transparent PNG on error)

## Chrome Extension

- MV3 extension installed at runtime by Puppeteer
- Configured via CDP Extensions.setStorageItems with `hostname` and `port`
- popup.js reads `chrome.storage.local` and fetches `/cost` to render a session cost table and total

## Technical Constraints

- Subresource policy:
  - Only navigation requests and image requests are forwarded to the proxy
  - All other subresources (e.g., external CSS/JS) are blocked
- CSS/JS handling:
  - External CSS/JS files are not fetched; generate inline CSS and JS within the streamed HTML
- Stateless navigations:
  - No persistent state between page loads; each navigation is independent
- Images and videos:
  - OpenRouter and Google/Gemini are supported; other selections fall back to a placeholder image
- Pricing and usage:
  - OpenRouter reports the exact per-request USD cost, which is used directly (Helicone lookup is skipped for openrouter)
  - For other providers, cost tracking is based on token usage reported by the Vercel AI SDK; some providers may not return usage, in which case costs default to 0 for that request
  - Pricing is fetched from Helicone per model; unknown models default to 0 cost
- Browser/runtime:
  - Headful Chrome is launched (not headless) to support extension install and devtools
  - Referer is stripped (set to empty) when forwarding requests to the proxy as a temporary workaround

## Dependencies (from package.json)

- ai
- @openrouter/ai-sdk-provider
- @ai-sdk/google
- @ai-sdk/openai
- @ai-sdk/anthropic
- @ai-sdk/groq
- puppeteer
- yargs
- dotenv
