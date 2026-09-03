# Using Titans from AI agents (MCP)

Every Titan speaks **MCP over stdio** through one shared server named
`titans`. The installer configures it automatically in the AI clients present
on the machine (Claude Code, Claude Desktop, VS Code, Codex, Antigravity);
`titans mcp configure --client <name> --yes` re-applies it any time.

- One server, every installed Titan: install Cronus later and its tools appear
  under the same `titans` server without touching client config again.
- Tools are namespaced (`titans_titan_atlas__atlas_search`,
  `titans_titan_cronus__cronus_job_submit`, ...).
- Payloads travel as canonical references (`atlas://tenant/project/...`),
  never inline bytes.
- The recommended agent pattern for long work: submit to Cronus, poll status,
  read results from Atlas — never block the conversation on a running job.

No ports, no API keys: stdio only, spawned locally by your client.

**Teach the agent first.** The skill [`skills/atlas-knowledge-graph.md`](../skills/atlas-knowledge-graph.md)
is the document that tells an AI how to use this product correctly; `titans skills show atlas-knowledge-graph`
prints it on any machine with the ToolManager.
