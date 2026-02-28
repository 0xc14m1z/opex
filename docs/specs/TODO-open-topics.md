# Open Topics — To Be Specced

## Critical (blocks implementation)

### 1. Pipeline Orchestrator
Who is the conductor? When Julius finishes decomposition, who triggers Sherlock for each
task? When all tasks complete, who tells Richelieu to open the PR? No agent currently owns
the end-to-end flow. Options: purely event-driven (each agent watches Redis for its input),
dedicated orchestrator agent, or one of the existing agents takes on the role.

**Status**: Next up for investigation.

### 2. Agent Prompts & Tool Definitions
The most important part of any agentic system. What system prompts does each agent get?
What tools (file read, shell exec, search) does each agent have in PydanticAI? How do we
version and iterate on prompts?

### 3. Model Selection Strategy
Nelson uses all 3 providers for consensus, but which model does Leonard use for its direct
implementation calls? Sherlock for analysis? Can agents use cheaper models for simpler
subtasks? Is there a routing strategy?

## Important (will cause problems later)

### 4. Rate Limiting
Nelson calls 3 providers in parallel. Multiple Leonards make direct calls too. LLM APIs
have rate limits. How do we manage this across the whole system?

### 5. Caching Strategy
Sherlock and Leonard may read the same files. Nelson may get similar prompts. Where do we
cache and how do we invalidate?

### 6. Nelson Concurrency
Katherine and Sherlock might both call Nelson simultaneously. Does Nelson queue requests,
run them in parallel, or have multiple instances?
