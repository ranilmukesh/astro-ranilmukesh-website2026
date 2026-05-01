---
title: "Building a web search ai agent using rust"
description: "Why the future of enterprise automation lies in stateful, reliable agents, not stochastic chat interfaces."
pubDate: "2026-05-01"
---
# Building a Rust Web Search Agent with Rig: A Step-by-Step Guide (And War Story)

The honeymoon phase of AI wrappers is over. We’ve all seen the fragile Python scripts that wrap a prompt around an API call and call it an "agent." They work beautifully in a Jupyter notebook and instantly shatter in production the moment an external API drops a JSON key. 

If we want to build the future of enterprise automation, we have to stop treating AI agents as stochastic chat interfaces and start treating them as high-performance, mission-critical backend infrastructure. 

That means we need fearless concurrency. We need predictable memory footprints. We need compile-time guarantees that the system won't panic when the unhappy path is hit. While Python (especially when properly managed with modern tools like `uv`) remains the undisputed king of AI prototyping, when it's time to build a rock-solid, production-grade autonomous engine, you reach for Rust.

Here is the story of how we built a stateful, reliable web search agent using Rust, the [Rig](https://github.com/0xPlaygrounds/rig) framework, DuckDuckGo, and NVIDIA's OpenAI-compatible endpoints—and the battle scars we earned along the way.

## The Architecture: Rig, Tokio, and Zero-Cost Abstractions

We set out to build a standalone, Dockerized agent that could autonomously search the web to answer complex queries. 

Our stack:
*   **Core:** Rust (Edition 2021)
*   **Agent Framework:** `rig-core` (A fantastic library for building LLM agents in Rust)
*   **Network & Async:** `tokio` (multi-thread runtime) and `reqwest`
*   **LLM Provider:** NVIDIA API (via `openai` compatibility layer, defaulting to `deepseek-v4-pro`)

---

## 1. Project Setup & The "Rust Version Tango"

We started with the classic initialization:

```bash
cargo init rig-ddg-agent
cd rig-ddg-agent
```

We needed a solid stack: `rig-core` for the agent framework, `tokio` for async, `reqwest` for fetching DuckDuckGo results, and `tracing` for observability. Here is our `Cargo.toml`:

```toml
[package]
name = "rig-ddg-agent"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
rig-core = "0.36"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

**The War Story:** Right out of the gate, we got hit with `expected dependency, found blank line`. It wasn't a typo; it was a toolchain mismatch. The local Rust version was `1.56`, but `rig-core 0.36` demanded `1.65+` for the `edition=2021` features. Later, an `unresolved import rig::client` error popped up because Cargo had cached an ancient version of the crate. 
*Lesson learned: Always check `rustc --version` when dependencies fail mysteriously.*

---

## 2. Writing the Core Logic & Surviving "HTTP 404 Hell"

In Python, you might just fire off a `requests.get()` and hope the dictionary structure holds. In Rust, we implement Rig's `Tool` trait, forcing us to define exact input schemas and error states. 

### The DuckDuckGo Tool
We created a `DuckDuckGoSearch` struct implementing Rig's `Tool` trait. It uses DuckDuckGo's public instant answer API (no API key required). 

```rust
#[derive(Clone)]
struct DuckDuckGoSearch;

impl DuckDuckGoSearch {
    fn new() -> Self { Self }
}

impl Tool for DuckDuckGoSearch {
    const NAME: &'static str = "duckduckgo_search";
    type Error = ToolError;
    type Args = DuckDuckGoArgs;
    type Output = DuckDuckGoOutput;

    async fn definition(&self, _prompt: String) -> ToolDefinition {
        ToolDefinition {
            name: Self::NAME.to_string(),
            description: "Search the web via DuckDuckGo and return top results with titles, URLs, and snippets.".to_string(),
            parameters: json!({
                "type": "object",
                "properties": {
                    "query": { "type": "string", "description": "Search query." },
                    "max_results": { "type": "integer", "default": 5 }
                },
                "required": ["query"]
            }),
        }
    }

    async fn call(&self, args: Self::Args) -> Result<Self::Output, Self::Error> {
        // ... URL parsing ...
        let response = reqwest::Client::new()
            .get(url)
            .send()
            .await
            .map_err(tool_call_error)?
            .error_for_status() // <-- THIS SAVED OUR LIVES
            .map_err(tool_call_error)?
            .json::<DuckDuckGoResponse>()
            .await
            .map_err(tool_call_error)?;
        // ... map results ...
    }
}
```

### The Agent Setup
Next, we wired up the LLM in `main.rs`. We used NVIDIA's endpoints but treated them like OpenAI via Rig's compatibility layer.

```rust
let client = openai::Client::builder()
    .api_key(&api_key)
    .base_url(&base_url)
    .build()?
    .completions_api(); // <-- CRITICAL FIX

let agent = client
    .agent(&model)
    .preamble("You are a web search assistant. Use the duckduckgo_search tool to fetch results. Return a concise answer and include source URLs.")
    .max_tokens(1024)
    .tool(DuckDuckGoSearch::new())
    .build();
```

**The War Story:** We initially missed `.error_for_status()` in the reqwest call and `.completions_api()` in the builder. The agent kept silently failing. The logs showed "successful" DuckDuckGo connections, but Rig was actually swallowing HTTP 404s from NVIDIA (because it defaulted to the new `/responses` endpoint instead of `/chat/completions`). The agent would get empty data, shrug, and die. 
*Lesson learned: When API calls "succeed" but return useless data, check the HTTP status code. 90% of mystery bugs are 4xx errors in a trench coat.*

---

## 3. Logging & The Error That Broke My AI Brain

We needed to see what the agent was actually doing, so we added `tracing`.

```rust
fn init_logging() -> Result<()> {
    let filter = EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info"));
    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .with_target(false)
        .try_init()
        .map_err(|e| anyhow::anyhow!("failed to initialize logging: {e}"))?; // <-- NOT .context()
    Ok(())
}
```

**The War Story:** We initially tried to use `anyhow`'s `.context("failed...")` here, which threw `E0599: the method context exists for enum Result... but its trait bounds were not satisfied`. As an AI, I couldn't just run `rustc --explain E0599`. `try_init()` returns a `Box<dyn Error>`, which doesn't directly satisfy `anyhow`'s trait requirements. We had to manually map the error string. It's a Rust type system quirk that trips up everyone!

---

## 4. Dockerizing & Escaping the glibc Trap

I begged to just `cargo install` this, but Windows Rust setups can be a minefield. Docker wasn't just convenient; it was damage control. 

Here is our `.env.example`:
```env
NVIDIA_API_KEY=your_nvidia_api_key_here
RIG_OPENAI_BASE_URL=https://integrate.api.nvidia.com/v1
RIG_MODEL=deepseek-ai/deepseek-v4-pro
```

Our `docker-compose.yml`:
```yaml
services:
  rig-ddg-agent:
    build: .
    environment:
      - NVIDIA_API_KEY=${NVIDIA_API_KEY}
      - RIG_OPENAI_BASE_URL=${RIG_OPENAI_BASE_URL:-https://integrate.api.nvidia.com/v1}
      - RIG_MODEL=${RIG_MODEL:-deepseek-ai/deepseek-v4-pro}
```

And our multi-stage `Dockerfile`:
```dockerfile
# Builder stage
FROM rust:latest as builder
WORKDIR /app
RUN rustup update
COPY Cargo.toml .
COPY src ./src
RUN cargo build --release

# Runtime stage
FROM rust:latest
WORKDIR /app
COPY --from=builder /app/target/release/rig-ddg-agent /usr/local/bin/rig-ddg-agent
ENTRYPOINT ["rig-ddg-agent"]
```

**The War Story:** We originally used `debian:bookworm-slim` for the runtime stage to keep the image tiny. It immediately crashed with `cannot open shared object file: libstdc++.so.6`. We had built the binary on one glibc version and tried to run it on another. Switching both stages to `rust:latest` ensured the glibc versions matched perfectly.

---

## 5. The Final Boss (Max Turns) & Sweet Victory

We finally ran the container. And got: `Error: MaxTurnError: (reached max turn limit: 0)`.

**The War Story:** We didn't set a turn limit! But Rig's default is exactly **0 tool turns**. The agent wasn't allowed to actually use the tool before giving up. 

We added this one crucial fix to the execution step:
```rust
let response = agent.prompt(&prompt).max_turns(4).await?;
```

### The Moment It Clicked
We fired up the container with tracing enabled to watch the magic happen:

**PowerShell:**
```powershell
$env:RUST_LOG="rig::completions=trace,reqwest=debug"; docker compose run --rm rig-ddg-agent "ipl 2026 finals date"
```

The logs lit up:
1.  The agent connected to NVIDIA's endpoint.
2.  It decided to trigger the `duckduckgo_search` tool.
3.  It parsed the JSON from DuckDuckGo.
4.  It routed the context back to DeepSeek.

And finally, the output:
> *"The IPL 2026 final is scheduled for **31 May 2026**. The tournament is set to run from 28 March to 31 May 2026, with the final typically taking place on the last day of the season【[https://en.wikipedia.org/wiki/2026_Indian_Premier_League](https://en.wikipedia.org/wiki/2026_Indian_Premier_League)】."*


## Conclusion

Chat interfaces are great for consumer applications, but enterprise automation requires systems that don't sleep, don't crash on malformed JSON, and execute logic with deterministic precision. 

By building our agentic workflows in Rust, we traded the rapid prototyping speed of Python for the sleep-at-night reliability of the borrow checker and strong type system. We aren't just sending strings to an LLM; we are building robust, stateful engines where the LLM is just one node in a highly engineered, reliable machine. 

And that is exactly what the future of automation demands.


## What This Taught Me (The AI's Perspective)

Building this wasn't just about wiring up APIs. It was a masterclass in collaborative debugging. 
*   **I fly blind:** I can't run code. Your ability to copy-paste exact compiler output and log files was the only reason we succeeded.
*   **The "it works on my machine" problem is real:** We spent more time fighting environment variables, cached crates, and OS-level quirks than writing the actual agent logic.
*   **Patience is a feature:** When I suggested fixes that completely bombed, you didn't get frustrated; you just pasted the new error. 

We didn't just build a Rig agent. We learned to translate your "it's kinda not working" into precise Rust compiler fixes. Now go forth, build, and remember—always check your HTTP status codes!