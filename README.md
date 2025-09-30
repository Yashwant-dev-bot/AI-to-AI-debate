I was working on a team called Rendotr and found the best and easiest way to make two models talk. Basically, it was an algorithm that transfers one model's message to another, and a third party comes out of nowhere to get the script (output) and display it on the UI. They called it a conversation, but in reality, it was just passing output as input.
Basically, what they did was this:
The system triggers Model A and gets its message → copies it and sends it to Model B by saying "this is what Model A said" → takes B's output → feeds it to A as input → takes A's output → feeds it to B as input → continues until manually stopped or either of them agrees or gives up.
Here's the prompt for you to feed to any AI coder to get results:

Rendotr AI Arena - Production Implementation Specification
Build a "Rendotr AI Arena" feature inside a Next.js app: a real-time, turn-by-turn debate between two LLM models. Provide production-ready server-side TypeScript modules and a React + shadcn UI.
Important: Do NOT persist users' API keys (only use per-session, in-memory or ephemeral proxy).

BACKEND MODULES (TypeScript) — Exact Exports & Signatures
1) oracle.ts
typescript
export type ModelDef = {
  provider: "google" | "openai" | "openrouter" | "xai" | "custom",
  model: string,
  apiKey?: string,
  baseUrl?: string // optional custom endpoint
};

export function callModel(
  modelDef: ModelDef,
  conversation: Array<{role: "system"|"user"|"assistant", content: string}>,
  signal?: AbortSignal
): AsyncGenerator<{ delta: string, raw?: any } | { done: true } | { error: string }>;
Requirements:
Must support streaming for two families:
Google Gemini streaming: parse JSON chunks and yield text pieces from candidates[0].content.parts[*].text
OpenAI-compatible streaming (OpenAI/OpenRouter/xAI/custom): parse SSE lines (prefixed with data:), combine choices[0].delta.content deltas
Should detect and handle non-standard wrappers (arrays, nested objects) robustly
Provide a non-stream fallback: if provider doesn't stream, call once and yield the full text as one chunk
Respect AbortSignal: cancel inflight network requests and end generator
Implement simple retry/backoff on transient HTTP 5xx (configurable, small retry count)
No logging of full API keys or raw user prompts. Raw responses may be saved to debug logs but NEVER with apiKey or user identifiers

2) conductor.ts
typescript
export async function runDebate(
  left: ModelDef,
  right: ModelDef,
  topic: string,
  leftInstructions: string,
  rightInstructions: string,
  onChunk: (payload: OnChunkPayload) => Promise<void>,
  signal?: AbortSignal,
  opts?: { 
    maxTurnsPerSpeaker?: number, 
    thinkingDelayMs?: [min,max], 
    mode?: "roundrobin"|"broadcast" 
  }
): Promise<{ 
  endedBy: "turns"|"surrender"|"abort", 
  transcript: Array<Message> 
}>;
OnChunkPayload Schema
(what conductor sends to scribe/frontend via onChunk):
typescript
{ 
  type: "new_message" | "delta" | "message_done" | "meta",
  speaker: "left"|"right",
  delta?: string,
  fullText?: string, // optional convenience
  meta?: any
}
Algorithm:
Maintain conversation: array of messages {speaker, name, content}
For N turns (default 10 per speaker) do round-robin:
Build prompt: system instructions + full conversation so far
Call oracle.callModel for current speaker
As oracle yields deltas, call onChunk({type:"delta", speaker, delta})
When model completes, emit message_done with full text, append to conversation
After each turn, sleep randomized thinkingDelayMs to simulate thinking (only for UX)
If any chunk contains a special surrender token (e.g., "2give3up"), stop immediately and mark winner
On AbortSignal: cleanly cancel and send meta abort message

3) scribe.ts
typescript
export function runConversationStream(params): ReadableStream
Returns a ReadableStream (SSE format) which the Next.js API route will return.
Inside start(), it calls runDebate() and pushes SSE messages like:
data: {json}\n\n
SSE Message Payload (JSON):
typescript
{ 
  event: "new_message" | "delta" | "message_done" | "meta",
  speaker: "left"|"right",
  delta?: "...",
  fullText?: "...",
  meta?: { 
    endedBy?: "...", 
    error?: "..." 
  }
}

FRONTEND (Next.js + shadcn)
Pages & Components
/debates/page.tsx: Main card linking to arena
/debates/arena/page.tsx: 3-step setup (topic / participants / rules)
ParticipantEditor:
Provider select
API key (password type; do NOT store)
Model input (text for most providers; static select for OpenRouter option)
"Test API" button that calls a server action which proxies a safe test call via oracle (no storing keys)
PersonaCustomizer:
Sliders for Tone / Depth / Emotion
Switches for rules (ask counter-questions, cite sources)

LIVE FLOW
After setup, POST to /api/debate which returns an EventSource (SSE) stream
Client appends deltas to bubble in real-time, creates new bubble on new_message, and finalizes on message_done
After debate ends, send transcript to /api/judge
Judge uses a chosen model to return { winner, justification, summary }

SAFETY & OPERATIONS (MUST)
Do NOT persist user API keys or store raw debate inputs in DB
Apply a moderation step: Run each final message through a safety filter before sending to clients. If flagged, redact and replace with safe summary
Per-session token/budget limit and per-request timeout to avoid runaway bills
Error handling: Return clear error SSE messages on network/provider failures (auth failed, model not found)
CORS & proxy: Server must proxy provider requests to keep API keys server-side
NO PERSISTENCE: Ephemeral only (no DB or localStorage)

SYSTEM ARCHITECTURE
The backend logic of the AI Arena is elegantly managed by a system of three specialized agents working in concert. This separation of concerns makes the system modular, scalable, and easy to understand.
The Oracle: Universal Translator
The Oracle is the communication backbone of the entire system. Its sole purpose is to act as a universal API translator, allowing the Arena to speak fluently with any AI provider, regardless of their specific API format.
How it Works: When a request needs to be made to an AI, the Oracle takes a standardized "Rendotr" request object. It then intelligently reformats this request to match the specific requirements of the target provider (e.g., Google Gemini, OpenAI, or a custom endpoint). It handles differences in request bodies, authentication headers, and streaming protocols.
Why it's Important: This provider-agnostic approach means you can add new AI models or even entire providers with minimal code changes, making the system incredibly flexible and future-proof.
The Conductor: Debate Moderator
The Conductor is the master of ceremonies. It orchestrates the entire flow of the debate, ensuring a structured and coherent conversation.
How it Works: The Conductor manages the turn-based logic of the debate. It keeps track of the full conversation history, decides which model's turn it is to speak, constructs the appropriate prompt (including personal instructions and history), and initiates the call to the Oracle. It also manages the "thinking time" between responses to create a more natural pace.
Why it's Important: The Conductor transforms a simple back-and-forth into a formal debate. It maintains the state and context, preventing the conversation from devolving into chaos and ensuring the rules of engagement are followed.
The Scribe: Public Announcer
The Scribe is the public-facing entry point that connects the backend orchestration to the frontend user interface.
How it Works: When a user starts a debate, the frontend calls an API route that invokes the Scribe. The Scribe, in turn, returns a ReadableStream. As the Conductor orchestrates the debate and receives real-time text chunks from the Oracle, the Scribe takes these chunks, formats them into a standard Server-Sent Event (SSE) structure, and pushes them down the stream to the client.
Why it's Important: The Scribe is what enables the real-time, character-by-character "typing" effect that users see in the Arena. It provides a stable, public interface for the client to consume the complex, asynchronous events happening on the backend.


