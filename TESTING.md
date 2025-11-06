# AFv2 Pattern #2: Parallel - Test Cases

## Overview

**Pattern:** Multi-source concurrent information gathering with aggregation and conflict resolution
**Flow:** Start → [WebSearch || KnowledgeBase || Analyzer] → Aggregator → Report → Direct Reply
**Repository:** https://github.com/snedea/afv2-pattern-02-parallel

---

## Prerequisites

### 1. Import Pattern into Flowise

1. Open Flowise UI (http://localhost:3000)
2. Navigate to **Agentflows** section
3. Click **"Add New"** → **"Import Agentflow"**
4. Upload `02-parallel.json`
5. Pattern should load with 10 nodes:
   - Start
   - Agent.WebSearch (Branch A)
   - Agent.KnowledgeBase (Branch B)
   - Agent.Analyzer (Branch C)
   - Agent.Aggregator
   - Agent.Report
   - Direct Reply
   - 3 Sticky Notes (documentation)

### 2. Configure API Keys

**All 5 agents require Anthropic API key configuration:**

1. Click on **Agent.WebSearch** node
2. In the **Model** dropdown, select model configuration:
   - **Credential:** Select your "Anthropic API Key" credential
   - **Model Name:** `claude-sonnet-4-5-20250929`
   - **Temperature:** `0.3` (for WebSearch/KnowledgeBase)
   - **Streaming:** `true` (default)

3. Repeat for:
   - **Agent.KnowledgeBase** (temperature: 0.3)
   - **Agent.Analyzer** (temperature: 0.2)
   - **Agent.Aggregator** (temperature: 0.2)
   - **Agent.Report** (temperature: 0.2)

4. **IMPORTANT:** Enable **Web Search Built-in Tool** for Agent.WebSearch:
   - Click **Agent.WebSearch** node
   - Locate **"Anthropic Built-in Tools"** section
   - Check **"Web Search"** (`web_search_20250305`)

5. Save the workflow

### 3. Verify Node Configuration

**Expected Configuration:**

| Node | Type | Tools | State Updates | Key Feature |
|------|------|-------|---------------|-------------|
| Agent.WebSearch | Agent | `web_search_20250305`, `currentDateTime` | `branches.web.results`, `branches.web.completed=true` | External web search |
| Agent.KnowledgeBase | Agent | `currentDateTime` | `branches.kb.results`, `branches.kb.citations`, `branches.kb.completed=true` | Internal KB/RAG |
| Agent.Analyzer | Agent | `calculator`, `currentDateTime` | `branches.analysis.results`, `branches.analysis.completed=true` | Structured analysis |
| Agent.Aggregator | Agent | `currentDateTime` | `aggregate.summary`, `aggregate.citations`, `aggregate.conflicts_resolved` | Dedup + conflict resolution |
| Agent.Report | Agent | `currentDateTime` | `report.generated`, `report.timestamp` | Final report generation |
| Direct Reply | DirectReply | N/A | N/A | Terminal node (hideOutput: true) |

---

## Test Cases

### TC-2.1: Basic Parallel Execution (Happy Path)

**Objective:** Verify all 3 branches execute concurrently and Aggregator waits for completion

**Input:**
```
Research the current state of quantum computing. Provide an analysis of recent breakthroughs, industry adoption, and projected timelines for practical applications.
```

**Expected Execution Flow:**

1. **All 3 Branches Execute in Parallel:**

   **Branch A: Agent.WebSearch**
   - Uses `web_search_20250305` tool to search external sources
   - Gathers recent news, articles, research papers
   - Updates state:
     ```json
     {
       "branches.web.results": "[web search findings]",
       "branches.web.completed": "true"
     }
     ```

   **Branch B: Agent.KnowledgeBase**
   - Queries internal knowledge base or RAG system
   - Retrieves documented facts with citations
   - Updates state:
     ```json
     {
       "branches.kb.results": "[KB findings]",
       "branches.kb.citations": "[source references]",
       "branches.kb.completed": "true"
     }
     ```

   **Branch C: Agent.Analyzer**
   - Performs structured analysis using `calculator` tool
   - Calculates metrics, timelines, projections
   - Updates state:
     ```json
     {
       "branches.analysis.results": "[analysis findings]",
       "branches.analysis.completed": "true"
     }
     ```

2. **Agent.Aggregator Waits for All Branches:**
   - Checks state for:
     - `branches.web.completed = true`
     - `branches.kb.completed = true`
     - `branches.analysis.completed = true`
   - Only executes after ALL branches complete
   - Reads all branch results from state
   - Performs deduplication:
     - Removes duplicate facts from multiple sources
     - Ranks information by source reliability
   - Resolves conflicts:
     - Identifies contradictory information
     - Uses consensus/recency to resolve
   - Updates state:
     ```json
     {
       "aggregate.summary": "[unified summary]",
       "aggregate.citations": "[consolidated citations]",
       "aggregate.conflicts_resolved": "[conflict resolution notes]"
     }
     ```

3. **Agent.Report Executes:**
   - Reads `aggregate.*` from state
   - Generates structured report with:
     - Executive summary
     - Findings from each branch
     - Consolidated citations
     - Conflict resolution notes
   - Updates state: `report.generated=true`, `report.timestamp`

4. **Direct Reply Returns:**
   - Displays completion message with comprehensive report

**Validation Checklist:**

- [ ] **Parallel Execution:**
  - All 3 branches (WebSearch, KnowledgeBase, Analyzer) executed
  - Branches executed concurrently (not sequentially)
  - Each branch completed independently

- [ ] **State Updates (Branch A - WebSearch):**
  - State contains `branches.web.results`
  - State contains `branches.web.completed=true`
  - Web search tool was invoked (check logs)

- [ ] **State Updates (Branch B - KnowledgeBase):**
  - State contains `branches.kb.results`
  - State contains `branches.kb.citations`
  - State contains `branches.kb.completed=true`

- [ ] **State Updates (Branch C - Analyzer):**
  - State contains `branches.analysis.results`
  - State contains `branches.analysis.completed=true`
  - Calculator tool was invoked if needed

- [ ] **Aggregator Execution:**
  - Aggregator executed ONLY after all 3 branches completed
  - Aggregator accessed all branch results from state
  - State contains `aggregate.summary`
  - State contains `aggregate.citations`
  - State contains `aggregate.conflicts_resolved`

- [ ] **Deduplication:**
  - Duplicate facts removed from aggregate.summary
  - Information ranked by source reliability
  - Final summary is shorter than sum of branch results

- [ ] **Conflict Resolution:**
  - Contradictory information identified
  - Conflicts resolved using consensus/recency
  - `aggregate.conflicts_resolved` documents resolution strategy

- [ ] **Report Generation:**
  - Report includes findings from ALL 3 branches
  - Report includes consolidated citations
  - Report includes conflict resolution notes
  - State contains `report.generated=true`

- [ ] **Direct Reply:**
  - Displays comprehensive final report
  - Report is well-structured and readable

**Success Criteria:**
- All 3 branches executed in parallel (not sequential)
- Aggregator waited for ALL branches before executing
- Deduplication reduced redundant information
- Conflicts successfully identified and resolved
- Final report includes consolidated findings from all sources

**Timing Validation:**
```
Expected execution pattern:

00:00 - Start
00:00 - Branch A (WebSearch) starts
00:00 - Branch B (KnowledgeBase) starts
00:00 - Branch C (Analyzer) starts
[00:05 - Branch C completes]  // Fastest (no external calls)
[00:08 - Branch B completes]  // Medium (KB lookup)
[00:12 - Branch A completes]  // Slowest (web search)
00:12 - Aggregator starts (waits for all)
00:15 - Report executes
00:16 - Direct Reply

✅ PASS if: Aggregator starts AFTER all branches complete
❌ FAIL if: Aggregator starts before any branch completes
```

---

### TC-2.2: Conflict Resolution and Deduplication

**Objective:** Verify Aggregator correctly handles contradictory information from multiple sources

**Input:**
```
What is the current status of fusion energy research? Specifically, has net energy gain been achieved, and what are the projected timelines for commercial viability?
```

**Expected Behavior:**

Different branches may return contradictory information:
- **WebSearch:** "ITER project delayed to 2035" (recent news)
- **KnowledgeBase:** "ITER target completion 2025" (outdated documentation)
- **Analyzer:** "Based on historical data, projected 2030-2040" (calculation)

**Expected Execution Flow:**

1. **Branches Execute in Parallel:**
   - Each branch produces findings with potentially conflicting dates
   - All branches update their respective `branches.*.completed=true`

2. **Agent.Aggregator Identifies Conflicts:**
   - Detects contradictory information:
     - ITER completion: 2025 vs 2035
     - Net energy gain status: achieved (NIF 2022) vs not achieved
   - Documents conflicts in `aggregate.conflicts_resolved`:
     ```
     CONFLICT: ITER completion date
     - KB source (2020 docs): 2025
     - Web source (2024 news): 2035 (delayed)
     - RESOLUTION: Use web source (more recent)

     CONFLICT: Net energy gain achieved?
     - KB source: "Not yet achieved"
     - Web source: "NIF achieved in Dec 2022"
     - RESOLUTION: Use web source (more recent and specific)
     ```

3. **Deduplication:**
   - Multiple sources mention "fusion energy" (deduplicated)
   - Multiple sources mention "ITER project" (deduplicated)
   - Unique details from each source preserved

4. **Final Summary Produced:**
   - Prioritizes most recent information (web search)
   - Includes citations from all sources
   - Documents conflict resolution strategy
   - Produces unified answer: "Net energy gain achieved (NIF 2022), ITER delayed to 2035"

**Validation Checklist:**

- [ ] **Conflict Detection:**
  - Aggregator identified contradictory information
  - `aggregate.conflicts_resolved` contains conflict descriptions
  - Each conflict shows source information (web vs KB vs analysis)

- [ ] **Conflict Resolution Strategy:**
  - Recency prioritized (newer info preferred)
  - Consensus used when dates unavailable
  - Resolution strategy documented

- [ ] **Deduplication:**
  - Duplicate facts removed (not repeated)
  - Unique information from each source preserved
  - Final summary shorter than combined branch results

- [ ] **Source Tracking:**
  - Each fact attributed to source (web/KB/analysis)
  - Citations preserved in `aggregate.citations`
  - Conflicting sources both documented

- [ ] **Final Summary Quality:**
  - Unified answer (not contradictory)
  - Most accurate/recent information used
  - Conflict resolution transparent to user

**Success Criteria:**
- Contradictory information successfully identified
- Conflicts resolved using documented strategy (recency/consensus)
- Final summary is consistent and unified
- All sources properly cited
- User receives clear, non-contradictory answer

**Example Expected Output:**
```
✅ CORRECT:
aggregate.summary: "Net energy gain achieved by NIF in December 2022. ITER project delayed to 2035 (originally 2025). Commercial viability projected 2030-2040."

aggregate.conflicts_resolved: "CONFLICT: ITER timeline - KB (2025) vs Web (2035) → Resolved using web source (more recent)"

❌ INCORRECT:
aggregate.summary: "ITER completion 2025. ITER completion 2035." [Contradictory!]
aggregate.conflicts_resolved: "" [No conflict resolution!]
```

---

### TC-2.3: Parallel Completion Timing and State Synchronization

**Objective:** Verify Aggregator correctly waits for ALL branches before executing

**Input:**
```
Compare three cloud providers: AWS, Azure, and Google Cloud. Focus on pricing, performance, and enterprise features.
```

**Expected Execution Flow:**

1. **Branches Execute with Different Completion Times:**
   - **WebSearch:** Slowest (10-15 seconds, external API calls)
   - **KnowledgeBase:** Medium (5-8 seconds, internal lookup)
   - **Analyzer:** Fastest (3-5 seconds, no external calls)

2. **State Updates at Different Times:**

   ```
   T=00:00 - Start
   T=00:00 - All branches start concurrently

   T=00:04 - Analyzer completes first
   State: {
     "branches.analysis.results": "[analysis complete]",
     "branches.analysis.completed": "true",
     "branches.web.completed": undefined,     // ❌ Not complete
     "branches.kb.completed": undefined        // ❌ Not complete
   }
   Aggregator: Does NOT execute (waiting for web + KB)

   T=00:07 - KnowledgeBase completes second
   State: {
     "branches.analysis.completed": "true",   // ✅ Complete
     "branches.kb.results": "[KB results]",
     "branches.kb.completed": "true",         // ✅ Complete
     "branches.web.completed": undefined       // ❌ Not complete
   }
   Aggregator: Does NOT execute (waiting for web)

   T=00:12 - WebSearch completes last
   State: {
     "branches.analysis.completed": "true",   // ✅ Complete
     "branches.kb.completed": "true",         // ✅ Complete
     "branches.web.results": "[web results]",
     "branches.web.completed": "true"         // ✅ Complete
   }
   Aggregator: NOW executes (all branches complete)
   ```

3. **Aggregator Waits for All Branches:**
   - Checks state for `branches.*.completed = true` for ALL branches
   - Does NOT proceed until ALL 3 flags are `true`
   - Only accesses branch results AFTER all branches complete

**Validation Checklist:**

- [ ] **Concurrent Execution:**
  - All 3 branches started at T=0 (not sequential)
  - Branches executed independently (not blocking each other)
  - Timestamps show concurrent execution

- [ ] **Completion Order:**
  - Analyzer completed first (fastest, no external calls)
  - KnowledgeBase completed second (medium, internal lookup)
  - WebSearch completed last (slowest, external API)

- [ ] **State Synchronization:**
  - Each branch set `branches.*.completed=true` on completion
  - State tracked completion for all 3 branches independently
  - No race conditions or missing state updates

- [ ] **Aggregator Wait Logic:**
  - Aggregator did NOT execute when only Analyzer complete
  - Aggregator did NOT execute when only Analyzer + KB complete
  - Aggregator ONLY executed when ALL 3 branches complete
  - **CRITICAL:** Verify Aggregator execution timestamp > latest branch completion time

- [ ] **No Premature Execution:**
  - Aggregator did not start before all branches completed
  - No partial results processed
  - All branch results available when Aggregator started

**Success Criteria:**
- All 3 branches executed concurrently (not sequentially)
- Branches completed in expected order (Analyzer → KB → WebSearch)
- Aggregator waited for ALL branches before executing
- State correctly tracked completion status for each branch
- No race conditions or missing data

**Timing Validation Script:**
```javascript
// Expected execution pattern
const execution = {
  start: 0,
  branches: {
    analyzer: { start: 0, end: 4 },     // Completes first
    kb: { start: 0, end: 7 },           // Completes second
    web: { start: 0, end: 12 }          // Completes last
  },
  aggregator: { start: 12, end: 15 },   // Starts AFTER all branches
  report: { start: 15, end: 16 },
  directReply: { start: 16, end: 16 }
};

// Validation rules
assert(aggregator.start >= Math.max(
  branches.analyzer.end,
  branches.kb.end,
  branches.web.end
)); // ✅ Aggregator waits for ALL branches

assert(branches.analyzer.start === branches.kb.start === branches.web.start); // ✅ Concurrent start
```

**State Validation:**
```bash
# After Analyzer completes (T=4s)
curl -X GET "http://localhost:3000/api/v1/agentflows/state" | jq '.branches'
# Expected: { "analysis": { "completed": "true" } }
#           { "web": {}, "kb": {} }  // ❌ Not complete yet

# After KB completes (T=7s)
curl -X GET "http://localhost:3000/api/v1/agentflows/state" | jq '.branches'
# Expected: { "analysis": { "completed": "true" }, "kb": { "completed": "true" } }
#           { "web": {} }  // ❌ Not complete yet

# After WebSearch completes (T=12s)
curl -X GET "http://localhost:3000/api/v1/agentflows/state" | jq '.branches'
# Expected: { "analysis": { "completed": "true" }, "kb": { "completed": "true" }, "web": { "completed": "true" } }
#           ✅ ALL complete, Aggregator can proceed
```

---

## Common Issues & Debugging

### Issue 1: Web Search Tool Not Available

**Symptoms:**
- Agent.WebSearch fails with "tool not found" error
- Web search results are empty or missing

**Debugging Steps:**
1. Verify Anthropic Built-in Tools are enabled:
   - Click **Agent.WebSearch** node
   - Check **"Anthropic Built-in Tools"** section
   - Ensure **"Web Search"** (`web_search_20250305`) is checked

2. Verify API key has web search access:
   - Some Anthropic API keys may not have web search enabled
   - Check Anthropic Console for feature availability

3. Check Flowise version supports built-in tools:
   - Web search requires Flowise v2.0+
   - Update Flowise if using older version

**Solution:**
- Enable "Web Search" in Agent.WebSearch configuration
- Verify API key has web search access
- Update Flowise to latest version

---

### Issue 2: Aggregator Executes Before Branches Complete

**Symptoms:**
- Aggregator runs but produces incomplete results
- Missing data from one or more branches
- State shows some `branches.*.completed` are undefined

**Debugging Steps:**
1. Verify each branch updates `branches.*.completed=true`:
   - **WebSearch:** Should set `branches.web.completed=true`
   - **KnowledgeBase:** Should set `branches.kb.completed=true`
   - **Analyzer:** Should set `branches.analysis.completed=true`

2. Check Aggregator's wait logic:
   - Aggregator should check state for ALL 3 completion flags
   - Should NOT proceed until all flags are `true`

3. Verify edge connections:
   - Check edges from branches to Aggregator
   - Ensure Aggregator has 3 incoming edges (one from each branch)

**Solution:**
- Add state updates for `branches.*.completed=true` to each branch
- Configure Aggregator to wait for all completion flags
- Verify 3 separate edges connect branches to Aggregator

---

### Issue 3: Branches Execute Sequentially (Not in Parallel)

**Symptoms:**
- Branches execute one after another (not concurrently)
- Total execution time = sum of branch times (not max)
- Timestamps show sequential execution

**Debugging Steps:**
1. Verify edge configuration from Start node:
   - Start should have 3 separate outgoing edges
   - Each edge targets one branch (WebSearch, KB, Analyzer)
   - Check `edges` array in JSON:
     ```json
     [
       { "source": "startAgentflow_0", "target": "agent_web_search" },
       { "source": "startAgentflow_0", "target": "agent_kb" },
       { "source": "startAgentflow_0", "target": "agent_analyzer" }
     ]
     ```

2. Check for accidental sequential edges:
   ```bash
   # Should NOT have these edges:
   jq '.edges[] | select(.source == "agent_web_search" and .target == "agent_kb")' 02-parallel.json
   jq '.edges[] | select(.source == "agent_kb" and .target == "agent_analyzer")' 02-parallel.json
   # Should return NO results
   ```

**Solution:**
- Ensure Start has 3 separate outgoing edges (fan-out pattern)
- Remove any sequential edges between branches
- Verify parallel execution in Flowise UI (all branches should show "running" simultaneously)

---

### Issue 4: Deduplication Not Working

**Symptoms:**
- Final summary contains duplicate facts
- Information repeated from multiple sources
- Summary is longer than expected

**Debugging Steps:**
1. Verify Aggregator receives ALL branch results:
   - Check state contains `branches.web.results`, `branches.kb.results`, `branches.analysis.results`
   - Aggregator should read all 3 results from state

2. Check Aggregator prompt includes deduplication instructions:
   - Prompt should explicitly instruct to remove duplicates
   - Should rank information by source reliability

3. Verify Aggregator has memory enabled:
   ```json
   {
     "agentEnableMemory": true  // ✅ Required for state access
   }
   ```

**Solution:**
- Enable memory for Aggregator
- Update Aggregator prompt with explicit deduplication instructions
- Verify Aggregator accesses all branch results from state

---

### Issue 5: Conflicts Not Resolved or Documented

**Symptoms:**
- Final summary contains contradictory information
- `aggregate.conflicts_resolved` is empty
- No conflict resolution strategy visible

**Debugging Steps:**
1. Verify Aggregator updates `aggregate.conflicts_resolved`:
   - Check state update configuration
   - Key: `aggregate.conflicts_resolved`
   - Value: `{{ conflicts }}`

2. Check Aggregator prompt includes conflict resolution instructions:
   - Should identify contradictory information
   - Should resolve using recency/consensus strategy
   - Should document resolution in output

**Solution:**
- Update Aggregator prompt with conflict resolution instructions
- Configure state update for `aggregate.conflicts_resolved`
- Test with inputs that produce known conflicts

---

## Test Execution Report

**Test Date:** `[YYYY-MM-DD]`
**Flowise Version:** `[version]`
**Pattern Version:** `1.0`

### Test Results Summary

| Test Case | Status | Duration | Notes |
|-----------|--------|----------|-------|
| TC-2.1: Basic Parallel Execution | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-2.2: Conflict Resolution | ⬜ Pass / ⬜ Fail | `[time]` | |
| TC-2.3: Parallel Timing | ⬜ Pass / ⬜ Fail | `[time]` | |

### Configuration Details

- **Anthropic Model:** `claude-sonnet-4-5-20250929`
- **Temperature (WebSearch/KB):** `0.3`
- **Temperature (Analyzer/Aggregator):** `0.2`
- **Web Search Enabled:** `Yes`
- **Streaming:** `Enabled`
- **Memory Enabled:** `Yes`

### Issues Encountered

`[List any issues encountered during testing]`

### Additional Notes

`[Any additional observations or recommendations]`

---

## Pattern Validation

This pattern has been validated against `validate_workflow.py` with the following checks:

✅ **Structural Validation:**
- Proper JSON structure with nodes and edges arrays
- All required node fields present
- All required edge fields present

✅ **Node Type Validation:**
- Start node with 3 outgoing edges (fan-out pattern)
- 5 Agent nodes with proper model configuration
- 1 DirectReply terminal node with hideOutput: true
- 3 StickyNote nodes for documentation

✅ **Parallel Execution:**
- Start has 3 separate edges to branches (WebSearch, KB, Analyzer)
- No sequential edges between branches
- Aggregator has 3 incoming edges (one from each branch)

✅ **State Management:**
- All agents have agentEnableMemory: true
- Branches update `branches.*.completed=true` on completion
- Aggregator reads all branch results from state
- Aggregator produces `aggregate.summary`, `aggregate.citations`, `aggregate.conflicts_resolved`

✅ **Tool Configuration:**
- WebSearch: `web_search_20250305`, `currentDateTime`
- KnowledgeBase: `currentDateTime`
- Analyzer: `calculator`, `currentDateTime`
- Aggregator: `currentDateTime`
- Report: `currentDateTime`

✅ **Edge Validation:**
- Parallel fan-out from Start (3 edges)
- Parallel fan-in to Aggregator (3 edges)
- Sequential flow: Aggregator → Report → Direct Reply

---

## Additional Resources

- **Pattern Library:** [Context Foundry AFv2 Patterns](https://github.com/snedea/afv2-patterns-index)
- **Pattern #2 Repository:** https://github.com/snedea/afv2-pattern-02-parallel
- **Flowise Documentation:** https://docs.flowiseai.com/
- **AgentFlow v2 Specification:** [AFv2 Docs](https://docs.flowiseai.com/agentflows)
- **Anthropic Web Search:** [Built-in Tools](https://docs.anthropic.com/claude/docs/tool-use#built-in-tools)

---

🤖 Built with Context Foundry
