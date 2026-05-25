# AWS LLM Wiki Schema

## Purpose
This document defines the rules for the AI agent maintaining this AWS Study Wiki. The goal is to build an interconnected graph of AWS services, concepts, and architectural patterns.

## Directory Structure
- `services/`: Specific AWS services (e.g., `EC2.md`, `S3.md`).
- `concepts/`: Broad cloud and system design concepts (e.g., `High Availability.md`, `Serverless.md`).
- `architectures/`: Design patterns and reference architectures.
- `certifications/`: Exam objectives and domain breakdowns.
- `raw/`: Unprocessed study material for the agent to ingest.

## Core Rules for the Agent
1. **Frontmatter**: Every new note MUST contain YAML frontmatter:
   ```yaml
   ---
   tags: []
   aliases: []
   date: YYYY-MM-DD
   ---
   ```
2. **Wiki-linking**: ALWAYS use Obsidian-compatible `[[Wiki-links]]` to reference other services, concepts, or architectures. Proactively link mentions of AWS services like `[[S3]]` or `[[EC2]]`.
3. **Log Execution**: Whenever you ingest material or create notes, append a brief entry to `log.md`.
4. **Index Update**: Ensure any major new notes are added to the appropriate section in `index.md`.
