# LlamaManager 0.2 Plan Index

The approved design is split into four implementation plans so each subsystem can be reviewed and shipped independently:

1. `2026-09-15-llamamanager-0.2-web-security.md`
   - visual redesign;
   - Auto / Light / Dark themes;
   - browser-local theme state;
   - CSRF protection and safe Hugging Face token handling.

2. `2026-09-15-llamamanager-0.2-mcp-localization.md`
   - optional MCP server;
   - Agent privilege boundary;
   - read-only/full-control policy;
   - dangerous operation flags;
   - temporary localization Developer Mode;
   - Russian localization completion.

3. `2026-09-15-llamamanager-0.2-cluster-placement-benchmarks.md`
   - LlamaNode identity and pairwise trust;
   - Cluster Gateway;
   - Brain state / host inspection;
   - placement and fit rules;
   - non-disruptive benchmark evidence;
   - MCP and Web cluster views;
   - optional mDNS discovery.

4. `2026-09-15-llamamanager-0.2-ai-skills-release.md`
   - final MCP surface integration;
   - Web/localization completion;
   - systemd installer integration;
   - documentation and prerelease packaging.

All four plans implement the approved architecture in `docs/superpowers/specs/2026-09-15-llamamanager-mcp-theme-design.md`.
