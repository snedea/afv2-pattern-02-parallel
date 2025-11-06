# Integration Guide: Pattern #2 - Parallel

**Pattern Type**: Concurrent Multi-Source
**Difficulty**: Intermediate
**Estimated Setup Time**: 15-20 minutes

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Importing to Flowise](#importing-to-flowise)
3. [API Key Configuration](#api-key-configuration)
4. [Agent Configuration](#agent-configuration)
5. [Testing the Workflow](#testing-the-workflow)
6. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required

- **Flowise Instance**: Self-hosted or cloud deployment
  - Minimum version: Flowise 1.4.0+
  - Access to Flowise UI (e.g., `http://localhost:3000`)

- **Anthropic API Key**: Claude model access
  - Sign up at: https://console.anthropic.com/
  - API key format: `sk-ant-api03-...`
  - Recommended model: `claude-sonnet-4-5-20250929`

### Recommended

- **searXNG Instance** (for WebSearch agent):
  - Self-hosted: https://github.com/searxng/searxng
  - Public instance: https://searx.space (check availability)
  - Alternative: Disable WebSearch agent if not using web search

- **Knowledge Base** (for KnowledgeBase agent):
  - Internal documentation
  - Database access
  - File system with reference materials

### Optional

- **Monitoring Tools**: Track parallel execution performance
- **Rate Limiting**: Manage concurrent API calls

---

## Importing to Flowise

### Step 1: Download the Workflow

Ensure you have the `02-parallel.json` file from this repository.

### Step 2: Import to Flowise

1. Open Flowise UI in your browser
2. Navigate to **"Agentflows"** or **"Chatflows"** tab
3. Click **"Import"** button (top right)
4. Select `02-parallel.json` from your local filesystem
5. Wait for upload to complete

### Step 3: Verify Import

After import, you should see:

- **11 nodes** on the canvas:
  - 1 Start node (green)
  - 5 Agent nodes (blue): WebSearch, KnowledgeBase, Analyzer, Aggregator, Report
  - 1 Direct Reply node (teal)
  - 4 Sticky Note nodes (yellow) - documentation only

- **8 edges** showing parallel branching and convergence

**Visual Structure:**
```
                    ┌─→ WebSearch ──┐
                    │                │
Start ──────────────┼─→ KnowledgeBase ──→ Aggregator → Report → Direct Reply
                    │                │
                    └─→ Analyzer ───┘
```

**Key Pattern**: 3 parallel branches converge at Aggregator for synthesis.

---

## API Key Configuration

### Step 1: Add Anthropic Credential

1. In Flowise UI, navigate to **"Credentials"** (gear icon, left sidebar)
2. Click **"Add Credential"** → **"Anthropic"**
3. Enter:
   - **Name**: `anthropic-main` (or your preferred name)
   - **API Key**: Your `sk-ant-api03-...` key
4. Click **"Save"**

### Step 2: Assign Credential to Agents

Each of the 5 agent nodes needs the Anthropic credential:

| Agent Node | Purpose | Model Recommended |
|------------|---------|-------------------|
| **WebSearch** | Real-time web research | `claude-sonnet-4-5-20250929` |
| **KnowledgeBase** | Internal data retrieval | `claude-sonnet-4-5-20250929` |
| **Analyzer** | Data analysis | `claude-sonnet-4-5-20250929` |
| **Aggregator** | Merge parallel outputs | `claude-sonnet-4-5-20250929` |
| **Report** | Final formatting | `claude-sonnet-4-5-20250929` |

**For each agent node:**

1. Click the agent node to open configuration panel
2. Find **"Model"** dropdown
3. Select your Anthropic credential
4. Choose `claude-sonnet-4-5-20250929` model
5. Click **"Save"**

### Step 3: Configure searXNG (Optional)

If using the WebSearch agent with web search capabilities:

1. Navigate to **"Credentials"** → **"Add Credential"** → **"searXNG"**
2. Enter:
   - **Name**: `searxng-main`
   - **API Base**: Your searXNG instance URL (e.g., `https://searx.example.com`)
3. Click **"Save"**
4. In WebSearch agent configuration, assign the searXNG credential to the tool

---

## Agent Configuration

### WebSearch Agent (Parallel Branch 1)

**Role**: Performs real-time web research using searXNG

**Default Prompt:**
```
You are the WebSearch agent, specialized in real-time web research.
Your job: Search the web for current information related to the user's query.
Use the searXNG tool to find relevant, up-to-date sources.
Store your findings in Flow State as 'websearch_output'.
Focus on: recency, credibility, diversity of sources.
```

**Customization Options:**
- **Tools**:
  - `currentDateTime` (built-in) - provides temporal context
  - `searXNG` (configured) - web search capability
- **Search Strategy**:
  - Number of results: 5-10 per query
  - Source diversity: news, academic, forums
  - Recency filter: last 7 days for time-sensitive topics
- **Memory**: 5-10 messages (lightweight, focused on search)

**State Output:**
- Variable: `websearch_output`
- Format: JSON array of sources
- Example:
```json
{
  "sources": [
    {"title": "...", "url": "...", "snippet": "...", "date": "2025-11-06"},
    {"title": "...", "url": "...", "snippet": "...", "date": "2025-11-05"}
  ],
  "query_used": "user query refined for search",
  "results_count": 8,
  "search_timestamp": "2025-11-06T10:30:00Z"
}
```

**Performance**: ~3-7 seconds (depends on searXNG response time)

---

### KnowledgeBase Agent (Parallel Branch 2)

**Role**: Retrieves information from internal knowledge bases or documentation

**Default Prompt:**
```
You are the KnowledgeBase agent, specialized in retrieving internal information.
Your job: Search internal documentation, databases, or files for relevant data.
Use available tools to access knowledge repositories.
Store your findings in Flow State as 'kb_output'.
Focus on: internal policies, historical data, proprietary information.
```

**Customization Options:**
- **Tools**:
  - `currentDateTime` (built-in)
  - Database connectors (PostgreSQL, MongoDB, etc.)
  - File system access tools
  - Document retrieval (vector search, keyword search)
- **Knowledge Sources**:
  - Internal wikis or documentation systems
  - Company databases
  - File servers or cloud storage
- **Memory**: 10 messages (may need context from prior searches)

**State Output:**
- Variable: `kb_output`
- Format: JSON with retrieved documents/records
- Example:
```json
{
  "documents": [
    {"id": "DOC-123", "title": "...", "content": "...", "last_updated": "2025-10-15"},
    {"id": "DOC-456", "title": "...", "content": "...", "last_updated": "2025-09-20"}
  ],
  "source": "internal_wiki",
  "results_count": 2,
  "retrieval_method": "keyword_search"
}
```

**Performance**: ~2-5 seconds (depends on database query complexity)

**Note**: If you don't have internal knowledge bases, this agent can be:
- Repurposed for document analysis
- Removed from the workflow (see Troubleshooting)
- Configured to use public APIs or datasets

---

### Analyzer Agent (Parallel Branch 3)

**Role**: Performs computational analysis, calculations, or specialized processing

**Default Prompt:**
```
You are the Analyzer agent, specialized in data analysis and computation.
Your job: Analyze the user's query and provide analytical insights.
Perform calculations, trend analysis, or statistical processing as needed.
Store your analysis in Flow State as 'analyzer_output'.
Focus on: quantitative analysis, patterns, predictions, recommendations.
```

**Customization Options:**
- **Tools**:
  - `currentDateTime` (built-in)
  - Calculator tools (math operations)
  - Statistical analysis tools
  - Data visualization tools (optional)
- **Analysis Types**:
  - Quantitative: calculations, statistics, metrics
  - Qualitative: sentiment analysis, categorization
  - Predictive: trend forecasting, risk assessment
- **Memory**: 5 messages (focused on current analysis task)

**State Output:**
- Variable: `analyzer_output`
- Format: JSON with analysis results
- Example:
```json
{
  "analysis_type": "trend_analysis",
  "findings": {
    "trend": "increasing",
    "growth_rate": "15% per month",
    "confidence": 0.85
  },
  "methodology": "linear_regression",
  "data_points": 12,
  "recommendations": ["Continue monitoring", "Expand capacity"]
}
```

**Performance**: ~2-4 seconds (depends on computation complexity)

---

### Parallel Execution Behavior

**How Parallel Agents Work:**

1. **Start Node Triggers All 3 Agents Simultaneously**
   - WebSearch, KnowledgeBase, and Analyzer receive the same input
   - All 3 begin execution at the same time (no waiting)

2. **Independent Execution**
   - Each agent works independently (no coordination needed)
   - Each agent has its own tools and memory
   - Execution times may vary (5-7 seconds typical range)

3. **Aggregator Waits for All 3 to Complete**
   - Flowise waits until all parallel branches finish
   - Once all 3 outputs are in Flow State, Aggregator starts
   - Total parallel execution time = slowest agent's duration

**Performance Benefits:**
- **Sequential equivalent**: 9-16 seconds (3 agents × 3-5s each)
- **Parallel execution**: 5-7 seconds (max of the 3 durations)
- **Time savings**: ~40-60% faster than sequential

**Considerations:**
- **API Rate Limits**: 3 concurrent API calls to Anthropic (ensure quota allows)
- **Cost**: Same total cost as sequential (3 agents × token usage)
- **Reliability**: If one agent fails, Aggregator may wait indefinitely (see error handling below)

---

### Aggregator Agent (Convergence Point)

**Role**: Merges outputs from all 3 parallel agents, deduplicates, resolves conflicts

**Default Prompt:**
```
You are the Aggregator agent, responsible for synthesizing information from multiple sources.
You have access to 3 parallel outputs in Flow State:
- websearch_output: Real-time web research findings
- kb_output: Internal knowledge base results
- analyzer_output: Analytical insights

Your job:
1. Merge all information into a unified view
2. Deduplicate redundant information
3. Resolve conflicts (prioritize: internal KB > web > analysis for factual data)
4. Identify gaps or missing information
5. Store the aggregated result in 'aggregated_output'

Focus on: completeness, accuracy, coherence.
```

**Customization Options:**
- **Conflict Resolution Strategy**:
  - Source priority: KB > Web > Analyzer (default)
  - Recency-based: Newer information preferred
  - Consensus-based: Only include info confirmed by 2+ sources
- **Deduplication**:
  - Semantic similarity (use embeddings)
  - Exact match on titles/URLs
  - Remove redundant facts
- **Memory**: 15 messages (needs to see all parallel outputs)

**State Output:**
- Variable: `aggregated_output`
- Format: JSON with merged and deduplicated data
- Example:
```json
{
  "merged_sources": [
    {"info": "...", "sources": ["web", "kb"], "confidence": 0.95},
    {"info": "...", "sources": ["analyzer"], "confidence": 0.80}
  ],
  "conflicts_resolved": 2,
  "duplicates_removed": 5,
  "gaps_identified": ["missing cost estimate"],
  "source_summary": {
    "web_sources": 8,
    "kb_sources": 2,
    "analysis_insights": 3
  }
}
```

**Performance**: ~4-6 seconds (depends on volume of data to merge)

**Critical Role**: Aggregator is the bottleneck - ensure it has sufficient context window for all 3 inputs.

---

### Report Agent (Output Formatting)

**Role**: Generates final user-facing report from aggregated data

**Default Prompt:**
```
You are the Report agent, responsible for creating the final output.
You have access to 'aggregated_output' which contains merged information from:
- Web search results
- Internal knowledge base
- Analytical insights

Create a clear, structured report with:
- Executive summary
- Key findings from each source
- Recommendations or next steps
- Source attribution

Store the final report in 'report_output'.
```

**Customization Options:**
- **Format**: Markdown, JSON, HTML, plain text
- **Sections**:
  - Executive Summary (2-3 sentences)
  - Web Findings (with source links)
  - Internal Knowledge (with document references)
  - Analysis & Insights
  - Recommendations
  - Source Bibliography
- **Tone**: Professional, casual, technical (adjust for audience)

**State Output:**
- Variable: `report_output`
- Format: User-facing report (Markdown recommended)
- Example:
```markdown
# Research Report

## Executive Summary
Based on 8 web sources, 2 internal documents, and quantitative analysis,
the market is growing at 15% monthly with high confidence (0.85).

## Web Research Findings
- **Source 1**: [Title](URL) - Key insight...
- **Source 2**: [Title](URL) - Key insight...

## Internal Knowledge Base
- **DOC-123**: Company policy states...
- **DOC-456**: Historical data shows...

## Analysis & Insights
- Trend: Increasing
- Growth Rate: 15% per month
- Confidence: 85%

## Recommendations
1. Continue monitoring trends monthly
2. Expand capacity to handle 30% growth
3. Review policy DOC-123 for updates

## Sources
- Web: 8 sources (2025-11-05 to 2025-11-06)
- Internal KB: 2 documents
- Analysis: Linear regression on 12 data points
```

**Performance**: ~3-5 seconds

---

### Direct Reply Node (Terminal)

**Role**: Sends final output to user

**Configuration:**
- **Message Source**: Uses `{{report_output}}` from Flow State
- **Format**: Renders the report in chat interface

**No customization needed** - automatically displays Report agent's output.

---

## Testing the Workflow

### Step 1: Start a Test Session

1. In Flowise UI, open the imported chatflow
2. Click **"Test"** button (top right)
3. Chat interface opens

### Step 2: Sample Input

Enter a test query that benefits from multiple information sources:

**Example Input:**
```
Research the current state of AI agent frameworks:
- Latest developments from the web
- Our internal documentation on agent architecture
- Market growth analysis and trends
```

### Step 3: Expected Flow Execution

1. **Start node triggers** (~0.5 seconds):
   - Sends query to all 3 parallel agents simultaneously

2. **Parallel execution** (~5-7 seconds total):
   - **WebSearch**: Searches for recent AI agent framework news (5s)
   - **KnowledgeBase**: Retrieves internal agent docs (3s)
   - **Analyzer**: Analyzes market trends (4s)
   - All 3 run concurrently (total time = max = 5s, not sum = 12s)

3. **Aggregator executes** (after all 3 complete, ~5 seconds):
   - Waits for all 3 outputs to be available
   - Merges web results + KB docs + analysis
   - Deduplicates and resolves conflicts
   - Stores in `aggregated_output`

4. **Report executes** (~4 seconds):
   - Formats aggregated data into user-friendly report
   - Adds executive summary and recommendations
   - Stores in `report_output`

5. **Direct Reply displays** (~0.5 seconds):
   - Shows the final report in chat

**Total Duration**: ~15-17 seconds (parallel execution saves 5-9 seconds vs sequential)

### Step 4: Verify Outputs

Check Flow State (click **"Show State"** button) to see all intermediate outputs:

```json
{
  "websearch_output": {
    "sources": [
      {"title": "LangChain 0.1 Released", "url": "...", "date": "2025-11-05"},
      {"title": "AutoGen Framework Update", "url": "...", "date": "2025-11-04"}
    ],
    "results_count": 8
  },
  "kb_output": {
    "documents": [
      {"id": "ARCH-001", "title": "Agent Architecture Guide", "content": "..."}
    ],
    "results_count": 2
  },
  "analyzer_output": {
    "trend": "increasing",
    "growth_rate": "15% per month",
    "confidence": 0.85
  },
  "aggregated_output": {
    "merged_sources": [...],
    "conflicts_resolved": 2,
    "duplicates_removed": 5
  },
  "report_output": "# Research Report\n\n## Executive Summary\n..."
}
```

### Step 5: Performance Validation

Check execution times in Flowise logs or UI:
- Individual agent times should be visible
- Verify parallel agents didn't wait for each other
- Total time should be ~40-60% faster than if run sequentially

---

## Troubleshooting

### Issue 1: One Parallel Agent Hangs (Workflow Stalls)

**Symptoms:**
- Workflow starts, 2 agents complete quickly, 3rd never finishes
- Aggregator never executes
- Workflow appears frozen

**Solutions:**

1. **Check the slow agent's logs**:
   - Click the agent node → View Logs
   - Look for errors: API timeout, tool failure, invalid input

2. **Common causes**:
   - **WebSearch**: searXNG instance down or unreachable
     - Fix: Check searXNG URL, try public instance
   - **KnowledgeBase**: Database connection timeout
     - Fix: Verify credentials, check network
   - **Analyzer**: Infinite loop or excessive computation
     - Fix: Review prompt, add timeout logic

3. **Add timeout handling**:
   - In Flowise settings: `EXECUTION_TIMEOUT=30000` (30s per agent)
   - If agent exceeds timeout, it fails gracefully

4. **Disable problematic agent** (temporary workaround):
   - Delete the edge from Start → slow agent
   - Update Aggregator prompt to handle missing input
   - Example: "If websearch_output is missing, note it and continue"

---

### Issue 2: Aggregator Receives Partial Data

**Symptoms:**
- Aggregator executes but shows "missing websearch_output" or similar
- Report incomplete or shows errors

**Solutions:**

1. **Verify all 3 agents stored outputs correctly**:
   - Check Flow State for presence of:
     - `websearch_output`
     - `kb_output`
     - `analyzer_output`
   - If one is missing, that agent didn't complete successfully

2. **Check agent prompts explicitly store data**:
   - Each parallel agent MUST include:
     ```
     Store your findings in Flow State as 'websearch_output'.
     ```
   - Verify variable names match what Aggregator expects

3. **Review Aggregator's prompt**:
   - Make it robust to missing data:
     ```
     If any output is missing, note it and synthesize available data.
     ```

---

### Issue 3: Parallel Agents Return Duplicates

**Symptoms:**
- WebSearch and KnowledgeBase both return the same information
- Report shows redundant content

**Solutions:**

1. **Improve agent specialization**:
   - **WebSearch**: Focus on recency (last 7 days)
   - **KnowledgeBase**: Focus on internal-only information
   - **Analyzer**: Focus on quantitative insights

2. **Enhance Aggregator's deduplication**:
   - Update Aggregator prompt:
     ```
     Remove exact duplicates and semantically similar content.
     If web and KB both mention the same fact, keep KB version (more authoritative).
     ```

3. **Use similarity detection**:
   - Add embedding-based similarity check in Aggregator
   - Threshold: 0.90+ similarity = duplicate

---

### Issue 4: searXNG Tool Not Working

**Symptoms:**
- WebSearch agent returns empty results
- Error: "searXNG API unreachable"

**Solutions:**

1. **Verify searXNG credential**:
   - Credentials → searXNG → Check API Base URL
   - Try accessing URL in browser: `https://your-searxng.com`
   - Should return searXNG homepage

2. **Test searXNG directly**:
   ```bash
   curl "https://your-searxng.com/search?q=test&format=json"
   ```
   - Should return JSON with search results
   - If fails: searXNG instance is down or misconfigured

3. **Use public searXNG instance**:
   - Find active instance: https://searx.space
   - Update credential with public URL
   - Note: Public instances may have rate limits

4. **Fallback: Disable web search**:
   - Remove searXNG tool from WebSearch agent
   - Repurpose WebSearch agent for other tasks (e.g., API calls)

---

### Issue 5: API Rate Limit Exceeded (429 Error)

**Symptoms:**
- One or more parallel agents fail with "Rate limit exceeded"
- Error code: 429

**Solutions:**

1. **Anthropic API rate limits**:
   - Free tier: 5 requests/minute
   - Paid tier: 50+ requests/minute (varies by plan)
   - **Issue**: 3 parallel agents = 3 concurrent requests

2. **Check your API quota**:
   - Visit: https://console.anthropic.com/settings/limits
   - Verify current usage and limits

3. **Solutions**:
   - **Upgrade plan**: Increase rate limit
   - **Add retry logic**: Configure Flowise to retry on 429 (with exponential backoff)
   - **Sequential fallback**: If rate limits hit, disable parallel execution:
     - Change edges to sequential: Start → WebSearch → KnowledgeBase → Analyzer → Aggregator
     - Slower but more reliable under rate limits

---

### Issue 6: Flow State Variables Overwritten

**Symptoms:**
- Only one agent's output appears in Flow State
- Other outputs missing

**Solutions:**

1. **Verify unique variable names**:
   - WebSearch MUST use: `websearch_output` (not just `output`)
   - KnowledgeBase MUST use: `kb_output`
   - Analyzer MUST use: `analyzer_output`
   - If all use same name, they overwrite each other

2. **Check agent prompts**:
   - Each must explicitly state:
     ```
     Store your findings in Flow State as 'websearch_output'.
     ```

3. **Test individually**:
   - Temporarily disconnect 2 agents
   - Run just WebSearch → check if `websearch_output` appears
   - Reconnect and test next agent

---

### Issue 7: Report Shows Raw JSON Instead of Formatted Text

**Symptoms:**
- Direct Reply displays JSON object instead of readable report
- User sees: `{"report_output": "..."}`

**Solutions:**

1. **Check Direct Reply configuration**:
   - **Message** field should reference: `{{report_output}}`
   - If showing raw JSON, variable reference is incorrect

2. **Try explicit flow state reference**:
   - Update to: `{{$flow.report_output}}`

3. **Check Report agent output format**:
   - Report agent should output plain text or Markdown
   - Not JSON (unless that's desired format)

---

## Advanced Configuration

### Adjusting Parallel Branch Count

**Add More Parallel Agents** (e.g., 5 branches instead of 3):

1. Duplicate an existing agent node
2. Customize the new agent's persona and tools
3. Add edge from Start → new agent
4. Add edge from new agent → Aggregator
5. Update Aggregator prompt to expect new input variable

**Remove a Parallel Agent** (e.g., 2 branches instead of 3):

1. Delete the agent node (e.g., Analyzer)
2. Delete edges: Start → Analyzer, Analyzer → Aggregator
3. Update Aggregator prompt to remove references to `analyzer_output`

---

### Timeout Configuration

**Prevent hanging workflows**:

1. Set Flowise environment variable:
   ```env
   EXECUTION_TIMEOUT=30000  # 30 seconds per node
   ```

2. Add timeout to agent prompts:
   ```
   If you cannot complete the task in 25 seconds, return partial results.
   ```

3. Handle timeout gracefully in Aggregator:
   ```
   If any input is missing due to timeout, note it in the report.
   ```

---

### Error Handling Strategy

**Make parallel execution more robust**:

1. **Aggregator handles missing inputs**:
   ```
   Check which outputs are available:
   - If websearch_output missing: Note "Web search unavailable"
   - If kb_output missing: Note "Internal KB unavailable"
   - If analyzer_output missing: Note "Analysis unavailable"
   Continue with available data.
   ```

2. **Agent-level try/catch**:
   ```
   Try to search the web. If searXNG fails, return:
   {"error": "Search unavailable", "fallback": "manual search needed"}
   ```

3. **Monitoring**:
   - Log all agent start/end times
   - Alert if any agent takes >20 seconds
   - Track success rate per agent

---

### Related Patterns

**When to Use Different Patterns:**

- **Pattern #1 (Chaining)**: Use when steps MUST be sequential (output of step 1 needed for step 2)
- **Pattern #2 (Parallel)**: Use when sources are independent (web, KB, analysis can run simultaneously)
- **Pattern #3 (Routing)**: Use when only ONE source needed based on user intent
- **Pattern #6 (Hierarchy)**: Use when parallel agents need supervision or task delegation

**Combining Patterns:**

- **Parallel + Routing**: Route user query to appropriate parallel sub-workflow
- **Parallel + Chaining**: Use parallel gathering, then sequential processing
- **Parallel + Iteration**: Parallel research, then iterative refinement

---

## Success Criteria

You have successfully integrated Pattern #2 when:

- ✅ All 11 nodes render correctly in Flowise UI (7 functional + 4 sticky notes)
- ✅ Each of 5 agent nodes has Anthropic credential configured
- ✅ searXNG credential configured for WebSearch agent (if using web search)
- ✅ Test execution shows all 3 parallel agents running simultaneously
- ✅ Flow State shows all 5 outputs (websearch, kb, analyzer, aggregated, report)
- ✅ Total execution time 40-60% faster than sequential (15-17s vs 25-30s)
- ✅ Aggregator successfully merges and deduplicates data
- ✅ Final report displays in chat with formatted output

---

## Performance Benchmarks

### Expected Execution Times

| Stage | Duration | Notes |
|-------|----------|-------|
| Start → Parallel Agents | 5-7s | Max of 3 concurrent executions |
| Aggregator | 4-6s | Depends on data volume |
| Report | 3-5s | Formatting time |
| **Total** | **15-17s** | 40-60% faster than sequential |

### Sequential Comparison (if run one-by-one)

| Stage | Duration |
|-------|----------|
| WebSearch | 5-7s |
| KnowledgeBase | 2-4s |
| Analyzer | 3-5s |
| Aggregator | 4-6s |
| Report | 3-5s |
| **Total** | **25-30s** |

**Speedup**: Parallel execution saves 8-13 seconds (~45% faster)

---

## Next Steps

### Customize for Your Use Case

1. **Modify Parallel Agent Specializations**:
   - WebSearch: Focus on specific domains (news, academic, etc.)
   - KnowledgeBase: Connect to your internal systems
   - Analyzer: Add domain-specific analysis tools

2. **Adjust Aggregation Strategy**:
   - Change conflict resolution rules
   - Modify deduplication thresholds
   - Add quality scoring for sources

3. **Add More Parallel Branches**:
   - Social media monitoring
   - Competitor analysis
   - Regulatory compliance checks
   - Customer feedback analysis

4. **Optimize Performance**:
   - Cache frequent queries
   - Pre-fetch common knowledge base docs
   - Use faster models for non-critical agents

### Explore Related Patterns

- **Pattern #1 (Chaining)**: Sequential pipeline when order matters
- **Pattern #3 (Routing)**: Dynamic agent selection based on intent
- **Pattern #6 (Hierarchy)**: Supervisor orchestrating parallel workers
- **Pattern #10 (RAG)**: Document retrieval with citations

### Production Readiness Checklist

- [ ] Test with real queries (not just samples)
- [ ] Verify all 3 parallel agents complete within timeout
- [ ] Add comprehensive error handling for each agent
- [ ] Configure API rate limits (3 concurrent requests to Anthropic)
- [ ] Set up monitoring for slow/failed agents
- [ ] Document searXNG instance failover strategy
- [ ] Test with missing/unavailable data sources
- [ ] Validate Aggregator deduplication works correctly
- [ ] Load test with 10+ concurrent sessions
- [ ] Benchmark execution time vs sequential baseline

---

## Support

**Pattern Issues**: https://github.com/snedea/afv2-pattern-02-parallel/issues
**Flowise Documentation**: https://docs.flowiseai.com/
**Pattern Index**: https://github.com/snedea/afv2-patterns-index

---

**Last Updated**: 2025-11-06
**Pattern Version**: 1.0
**Flowise Compatibility**: 1.4.0+
