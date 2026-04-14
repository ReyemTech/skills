---
name: teardown
description: Reverse-engineer any web app's architecture from its frontend. Dispatches to the teardown agent to keep main context clean.
disable-model-invocation: true
allowed-tools: Task
---

# Teardown -- Frontend Architecture Reverse Engineering

When the user invokes `/teardown`, spawn the `teardown` agent via Task() with the user's arguments.

## Usage

```
/teardown <url>
/teardown <url> --name codalio
/teardown <url> --output-dir path/to/output
/teardown <url> --name codalio --output-dir path/to/output
```

## Arguments

- `<url>` (required): Target site to analyze
- `--name <name>` (optional): Short name for the output file
- `--output-dir <path>` (optional): Directory to save results (defaults to `references/intelligence/`)

## Execution

Use Task() to run the `teardown` agent with the user's full arguments. The agent handles all phases:

1. HTTP headers reconnaissance
2. HTML source & JS bundle analysis
3. API discovery (REST, GraphQL, WordPress)
4. Analytics & third-party service identification
5. Compile and save results to the output directory

Wait for the agent to complete and return a summary of findings to the user.
