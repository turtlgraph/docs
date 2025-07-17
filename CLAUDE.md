# CLAUDE.md - GRAPHITE Documentation Project Guide

## Project Overview

This is the GRAPHITE (Graphite Recursive Asset Pipeline – Hypergraph Infinite Turtling Engine) documentation reorganization project. You are tasked with transforming a comprehensive 28-chapter, 500+ page technical specification into a well-structured, accessible documentation system.

## Current Status

### Completed
- ✅ Analyzed 4 organization options in `/docs/decisions/documentation-format.md`
- ✅ Selected Option D: Hybrid Technical/Practical Structure
- ✅ Created volume directory structure
- ✅ Archived original files in `/docs/archive/`
- ✅ Moved branding assets to `/marketing/branding/`

### In Progress
- 🔄 Creating volume-based documentation from archived sources
- 🔄 Integrating all 28 chapters into new structure

## Documentation Structure

The approved organization follows a 3-volume, 11-part structure:

### Volume 1: Foundation & Core Systems (Chapters 1-11)
- **Part 1**: Introduction & Architecture (Ch 1-3)
- **Part 2**: Data & Runtime (Ch 4-6)
- **Part 3**: System Services (Ch 7-9)
- **Part 4**: Platform & Security (Ch 10-11)

### Volume 2: Development & Integration (Chapters 12-14, 22-25)
- **Part 5**: Development Tools (Ch 12-14)
- **Part 6**: Integration & Migration (Ch 22-23, 25)
- **Part 7**: Real-World Application (Ch 24)

### Volume 3: Advanced Systems & Future (Chapters 15-21, 26-28)
- **Part 8**: Performance & Optimization (Ch 15-16)
- **Part 9**: Advanced Features (Ch 17-19)
- **Part 10**: Production & Analytics (Ch 20-21)
- **Part 11**: Ecosystem & Future (Ch 26-28)

## Source Files

All original content is preserved in `/docs/archive/`:
- `spec.md` - Clean v3.0 base (chapters 1-14)
- `part-3.md` - Chapters 15-19
- `part-4.md` - Chapters 20-24
- `part-5.md` - Chapters 25-28
- `claude-thoughts.md` - Philosophical insights on graph-native computing
- `gemini-report.md` - External evaluation
- Various `Untitled.md` files - Earlier versions and discussions

## Style Guide

### Technical Writing Principles
1. **Maintain Technical Depth**: This is a nuclear-grade technical specification. Never simplify at the expense of accuracy.
2. **Code Examples**: Include comprehensive C code examples as in the original
3. **Precise Language**: Use exact technical terminology
4. **Comprehensive Coverage**: Every detail matters - this could revolutionize game development

### Formatting Standards
1. **Headers**: Use clear hierarchical headers (# Volume, ## Part, ### Chapter, #### Section)
2. **Code Blocks**: Always specify language (```c, ```bash, ```typescript)
3. **Tables**: Use markdown tables for comparisons and specifications
4. **Lists**: Use bullet points for features, numbered lists for procedures
5. **Cross-References**: Link between related sections using relative paths

### Content Requirements
1. **Preserve ALL Content**: Every line from the 28 chapters must be included
2. **Add Navigation**: Each part needs:
   - Table of contents at the beginning
   - "Next/Previous" navigation at the end
   - Cross-references to related parts
3. **Include Diagrams**: Add Mermaid diagrams where they clarify architecture
4. **Integrate Insights**: Include philosophical insights from `claude-thoughts.md`

### File Naming Convention
```
/docs/volume-X-name/part-Y-title.md
```
Example: `/docs/volume-1-foundation/part-1-introduction-architecture.md`

## Implementation Instructions

### Step 1: Extract Content
For each part, extract the relevant chapters from the archive files:
1. Identify chapter boundaries (look for `## Section N:` patterns)
2. Copy complete chapter content including all code examples
3. Preserve all formatting and technical details

### Step 2: Structure Each Part
Each part file should contain:
```markdown
# Volume X: [Volume Title]
## Part Y: [Part Title]

### Table of Contents
- Link to each chapter/section

### Overview
Brief introduction to this part's content

### Chapter N: [Title]
[Complete chapter content]

### Chapter N+1: [Title]
[Complete chapter content]

### Cross-References
- Related concepts in Part Z
- Implementation details in Part W

### Navigation
[Previous: Part Y-1] | [Next: Part Y+1]
```

### Step 3: Add Enhancements
1. **Mermaid Diagrams**: Add where helpful, especially for:
   - Architecture overviews
   - Data flow diagrams
   - Bundle structure visualization
   - Graph relationships

2. **Practical Examples**: Integrate case studies at relevant points

3. **Implementation Notes**: Add notes about real-world usage

### Step 4: Validate Completeness
- Ensure all 28 chapters are present
- Verify no content was lost in reorganization
- Check all cross-references work
- Confirm code examples are complete

## Important Context

### The GRAPHITE Philosophy
GRAPHITE represents a paradigm shift: "Everything is a graph." This isn't just technical architecture—it's a recognition that:
- Git is actually a distributed graph database
- All computing abstractions are fundamentally graphs
- GRAPHITE is "Git for game assets"

### Key Innovation Points
1. **Graph-Native Design**: Assets and dependencies as first-class graph citizens
2. **Distributed Architecture**: Like Git, works offline with full functionality
3. **Universal Format**: Aims to be the industry standard
4. **Performance Focus**: Optimized for modern hardware (SSDs, GPUs)

### The GRAF Mascot
GRAF is the hexagonal graph mascot—a cyberpunk character on a skateboard with the GRAPHITE logo. Represents the fusion of technical excellence and developer culture.

## Quality Checklist

Before considering any part complete:
- [ ] All source content included verbatim
- [ ] Table of contents is accurate
- [ ] Code examples are properly formatted
- [ ] Cross-references are functional
- [ ] Navigation links work
- [ ] Technical accuracy maintained
- [ ] No details omitted or simplified

## Working Principles

1. **Comprehensive, Not Concise**: This documentation should be exhaustive
2. **Technical First**: Assume readers are experienced developers
3. **Future-Proof**: Structure allows for easy additions
4. **Graph-Centric**: Emphasize the graph nature of everything

## Next Actions

1. Start with Volume 1, Part 1 (Introduction & Architecture)
2. Extract Chapters 1-3 from `/docs/archive/spec.md`
3. Create `/docs/volume-1-foundation/part-1-introduction-architecture.md`
4. Follow the structure template above
5. Continue through all 11 parts systematically

Remember: This isn't just documentation—it's the technical foundation for what could become the game industry's next universal standard. Every detail matters. Make it legendary.

## Command Reminders

Useful commands for this project:
```bash
# Search for chapter boundaries
grep -n "^## Section" docs/archive/*.md

# Count lines in a file
wc -l docs/archive/spec.md

# Find specific content
grep -r "specific term" docs/archive/

# Preview markdown
# Use your markdown preview tool of choice
```

---

*"Where every asset is a graph, and every graph tells a story."*

**Status: Ready to transform 28 chapters into organized volumes**