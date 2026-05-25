# AWS Study Wiki & Knowledge Graph ☁️

An interconnected, Markdown-based study wiki and knowledge graph designed for mastering Amazon Web Services (AWS) concepts, services, and architectures—specifically optimized for the **AWS Certified Solutions Architect - Associate (SAA-C03)** exam.

Built for seamless integration with **Obsidian**, this repository uses bidirectional wiki-linking (`[[Wiki-links]]`) and YAML frontmatter to construct a visual, navigable web of AWS architectural patterns, services, and practice questions.

---

## 📂 Directory Structure

The repository is structured semantically to separate high-level services, deep-dive concepts, exam practice material, and supporting assets:

```text
├── services/          # Atomic, entry-level notes for core AWS services (e.g., EC2, VPC)
├── concepts/          # Deep-dive study notes on specific features, routing policies, or sub-systems
├── questions/         # Domain-specific practice questions with detailed answers and concept links
├── architectures/     # Design patterns and reference architectures
├── certifications/    # Study guides and domain breakdowns for specific exams
├── assests/           # Visual diagrams, charts, and architectural assets (referenced in notes)
├── raw/               # Staging area for raw materials awaiting ingestion or processing
├── index.md           # The primary Map of Content (MOC) serving as the wiki's home page
├── log.md             # Immutable changelog tracking the addition and refinement of notes
├── CLAUDE.md          # Rules and schema guidelines for AI agents updating this wiki
└── .gitignore         # Ignores IDE and Obsidian local workspace config files
```

---

## 🔑 Core Features & Design

### 🔗 Bidirectional Linking & Map of Content (MOC)
- **Map of Content (`index.md`)**: Serves as the homepage of the wiki, grouping topics by services, concepts, and certification objectives.
- **Interconnected Notes**: All notes utilize Obsidian-compatible `[[Wiki-links]]` to link to relevant parent/child topics. (e.g., [[RDS Read Replicas vs Multi-AZ]] links directly back to [[Amazon RDS]] and [[Amazon Aurora]]).

### 📊 Rich Visual Documentation
- Notes include detailed ASCII art architecture diagrams, comparative tables, and markdown-rendered callout blocks (`> [!TIP]`, `> [!WARNING]`, etc.) to highlight exam traps, cheat sheets, and triggers.
- Multi-dimensional comparison matrices (e.g., RDS Read Replicas vs Multi-AZ, S3 Storage Class transitions) simplify complex trade-offs.

### 📝 Integrated Quiz Engine (`questions/`)
- Contains hundreds of scenario-based practice questions.
- Quiz markdown files use standard checkboxes `[ ]` and `[x]` to mark correct answers, accompanied by deep-dive explanations referencing corresponding concept files.

---

## ⚙️ How to Use with Obsidian

To experience the interactive knowledge graph and clean rendering:

1. **Download & Install**: Install [Obsidian](https://obsidian.md/) (available on Windows, macOS, Linux, iOS, and Android).
2. **Open Vault**:
   - Open Obsidian and select **Open folder as vault**.
   - Navigate to and select this repository's root folder (`aws-`).
3. **Configure Settings (Recommended)**:
   - Go to **Settings** > **Files and links** > Enable **Use [[Wikilinks]]**.
   - Go to **Settings** > **Core plugins** > Enable **Graph view** and **Tag pane**.
4. **Navigate**:
   - Start at `index.md` (the MOC homepage).
   - Use `Ctrl + Click` (or `Cmd + Click`) on any `[[Link]]` to navigate to that note.
   - Click the **Graph view** icon in the sidebar to see the visual web of how services, concepts, and questions connect to one another.

---

## 🛠️ Contribution & Development Rules

If you are using an AI agent or manually adding new notes to the wiki, you must adhere to the rules defined in `CLAUDE.md`:

1. **Frontmatter**: Every new note MUST contain a YAML frontmatter block at the top containing tags, aliases (for search optimization), and the creation date:
   ```yaml
   ---
   tags: [concept, compute, storage, networking]
   aliases: [Alias 1, Alias 2]
   date: YYYY-MM-DD
   ---
   ```
2. **Wiki-linking**: Proactively link keywords to existing notes. Avoid plain text for AWS services if a note exists (e.g., use `[[S3]]` instead of `S3`).
3. **Log Changes**: Whenever notes are added or updated, append an entry detailing the changes to `log.md`.
4. **Update MOC**: Always add newly created files to the appropriate section of `index.md`.
