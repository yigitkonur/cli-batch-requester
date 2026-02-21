high-throughput batch API client for LLM workloads. load-balances across endpoints, retries with backoff, streams results to disk. written in Rust.

```
blaze -i requests.jsonl -o results.jsonl
```

that's it. 100k requests, zero babysitting.

[![crates.io](https://img.shields.io/crates/v/blaze-api.svg?style=flat-square)](https://crates.io/crates/blaze-api)
[![rust](https://img.shields.io/badge/rust-1.75+-93450a.svg?style=flat-square)](https://www.rust-lang.org/)
[![license](https://img.shields.io/badge/license-MIT-grey.svg?style=flat-square)](https://opensource.org/licenses/MIT)

---

## what it does

- **10k+ req/sec** on modest hardware (Tokio async runtime, work-stealing)
- **weighted load balancing** across multiple endpoints/API keys
- **exponential backoff with jitter** — respects rate limits, no thundering herd
- **streaming output** — results hit disk as they complete, crash-safe
- **per-endpoint health tracking** — unhealthy endpoints get cooled off automatically
- **connection pooling** — HTTP/2 keep-alive, no repeated TCP handshakes

## install

```bash
cargo install blaze-api
```

or build from source:

```bash
git clone https://github.com/yigitkonur/blaze-api.git
cd blaze-api && cargo build --release
```

also available via Homebrew on macOS:

```bash
brew install yigitkonur/tap/blaze
```

## usage

### basic

```bash
blaze -i requests.jsonl -o results.jsonl

# crank it up
blaze -i data.jsonl -o out.jsonl --rate 10000 --workers 200
```

### with multiple endpoints

```bash
blaze -i requests.jsonl -o results.jsonl --config endpoints.json
```

### via environment

```bash
export BLAZE_ENDPOINT_URL="https://api.openai.com/v1/completions"
export BLAZE_API_KEY="sk-..."
export BLAZE_MODEL="gpt-4"
blaze -i requests.jsonl -o results.jsonl
```

### input format

one JSON object per line:

```jsonl
{"input": "What is the capital of France?"}
{"input": "Explain quantum computing in simple terms."}
```

or with full request bodies:

```jsonl
{"body": {"messages": [{"role": "user", "content": "Hello!"}], "model": "gpt-4"}}
```

### output

successes go to your output file:

```jsonl
{"input": "...", "response": {"choices": [...]}, "metadata": {"endpoint": "...", "latency_ms": 234, "attempts": 1}}
```

failures go to `errors.jsonl`:

```jsonl
{"input": "...", "error": "HTTP 429: Rate limit exceeded", "status_code": 429, "attempts": 3}
```

## configuration

### CLI flags

```
OPTIONS:
    -i, --input <FILE>        JSONL input file
    -o, --output <FILE>       output file for successful responses
    -e, --errors <FILE>       error output [default: errors.jsonl]
    -r, --rate <N>            max requests/sec [default: 1000]
    -w, --workers <N>         concurrent workers [default: 50]
    -t, --timeout <SECS>      request timeout [default: 30]
    -a, --max-attempts <N>    retry attempts [default: 3]
    -c, --config <FILE>       endpoint config file (JSON)
    -v, --verbose             debug logging
        --json-logs           structured JSON logs
        --no-progress         disable progress bar
        --dry-run             validate without processing
```

all flags also work as env vars with `BLAZE_` prefix.

### endpoint config

for multiple endpoints with weighted distribution:

```json
{
  "endpoints": [
    {
      "url": "https://api.openai.com/v1/completions",
      "weight": 2,
      "api_key": "sk-key-1",
      "model": "gpt-4",
      "max_concurrent": 100
    },
    {
      "url": "https://api.openai.com/v1/completions",
      "weight": 1,
      "api_key": "sk-key-2",
      "model": "gpt-4",
      "max_concurrent": 50
    }
  ],
  "request": {
    "timeout": "30s",
    "rate_limit": 5000,
    "workers": 100
  },
  "retry": {
    "max_attempts": 3,
    "initial_backoff": "100ms",
    "max_backoff": "10s",
    "multiplier": 2.0
  }
}
```

## library usage

```rust
use blaze_api::{Config, EndpointConfig, Processor};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config {
        endpoints: vec![EndpointConfig {
            url: "https://api.example.com/v1/completions".to_string(),
            weight: 1,
            api_key: Some("your-key".to_string()),
            model: Some("gpt-4".to_string()),
            max_concurrent: 100,
        }],
        ..Default::default()
    };

    let processor = Processor::new(config)?;
    processor.process_file(
        "requests.jsonl".into(),
        Some("results.jsonl".into()),
        "errors.jsonl".into(),
        true,
    ).await?.print_summary();

    Ok(())
}
```

## troubleshooting

| problem | fix |
|:---|:---|
| "too many open files" | `ulimit -n 65535` |
| connection timeouts | increase `--timeout` or reduce `--workers` |
| 429 rate limit errors | lower `--rate` or add more API keys |
| high memory usage | reduce `--workers` |
| OpenSSL build errors | `apt install libssl-dev` or use `--features rustls` |

## project structure

```
src/
  lib.rs        — library entry point
  main.rs       — CLI
  config.rs     — configuration
  client.rs     — HTTP client + retry logic
  endpoint.rs   — load balancer
  processor.rs  — processing orchestration
  request.rs    — request/response types
  tracker.rs    — statistics
  error.rs      — error types
```

## contributing

```bash
cargo test && cargo fmt && cargo clippy
```

PRs welcome.

## license

MIT
