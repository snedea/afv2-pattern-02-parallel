# AFv2 Pattern #2: Parallel

Multi-source concurrent information gathering with aggregation and conflict resolution.

## Pattern Structure

```
Start → [Web Search || Knowledge Base || Analyzer] → Aggregator → Report → Direct Reply
```

## Key Features

- 3 concurrent branches executing in parallel
- Built-in web search tool integration
- Aggregation with deduplication and conflict resolution
- Direct Reply terminal node

## Files

- `02-parallel.json` - Complete Flowise workflow (874 lines)

## Quick Start

1. Import `02-parallel.json` into Flowise
2. Configure Anthropic API key for all agents
3. Test with research query

## Use Cases

- Research synthesis from multiple sources
- Competitive analysis (web + internal data + analysis)
- Risk assessment with parallel checks
- Multi-source fact verification

## Documentation

See [Context Foundry Pattern Library](https://github.com/context-foundry/context-foundry/tree/main/extensions/flowise/templates/afv2-patterns) for complete documentation.

🤖 Built with Context Foundry
