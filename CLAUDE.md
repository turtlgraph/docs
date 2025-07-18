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
- ✅ Created content mapping in `/docs/rewrite/mapping.md`
- ✅ Created implementation plan in `/docs/rewrite/plan.md`

### In Progress
- 🔄 Creating volume-based documentation from archived sources
- 🔄 Integrating all 28 chapters into new structure

## Chapter Progress Tracker

### Volume 1: Foundation & Core Systems
- [ ] Part 1: Introduction & Architecture (Ch 1-3)
  - [x] Chapter 1: Introduction & Overview - **Complete**
  - [x] Chapter 2: Core Concepts - **Complete**
  - [x] Chapter 3: Architecture Overview - **Complete**
- [x] Part 2: Data & Runtime (Ch 4-6)
  - [x] Chapter 4: Data Structures - **Complete**
  - [x] Chapter 5: File Format Specification - **Complete**
  - [x] Chapter 6: Runtime API - **Complete**
- [x] Part 3: System Services (Ch 7-9)
  - [x] Chapter 7: Memory Management - **Complete**
  - [x] Chapter 8: Streaming & Loading - **Complete**
  - [x] Chapter 9: Caching System - **Complete**
- [x] Part 4: Platform & Security (Ch 10-11)
  - [x] Chapter 10: Security & Encryption - **Complete**
  - [x] Chapter 11: Platform Abstraction Layer - **Complete**

### Volume 2: Development & Integration
- [ ] Part 5: Development Tools (Ch 12-14)
  - [ ] Chapter 12: Build System - **Not Started**
  - [ ] Chapter 13: Command Line Tools - **Not Started**
  - [ ] Chapter 14: Testing Framework - **Not Started**
- [ ] Part 6: Integration & Migration (Ch 22-23, 25)
  - [ ] Chapter 22: Editor Integration - **Not Started**
  - [ ] Chapter 23: Language Bindings - **Not Started**
  - [ ] Chapter 25: Migration & Compatibility - **Not Started**
- [ ] Part 7: Real-World Application (Ch 24)
  - [ ] Chapter 24: Real-World Case Studies - **Not Started**

### Volume 3: Advanced Systems & Future
- [ ] Part 8: Performance & Optimization (Ch 15-16)
  - [ ] Chapter 15: Performance Optimization - **Not Started**
  - [ ] Chapter 16: Advanced Integration Patterns - **Not Started**
- [ ] Part 9: Advanced Features (Ch 17-19)
  - [ ] Chapter 17: Virtual Bundle System - **Not Started**
  - [ ] Chapter 18: Graph Analysis Tools - **Not Started**
  - [ ] Chapter 19: Distributed Systems - **Not Started**
- [ ] Part 10: Production & Analytics (Ch 20-21)
  - [ ] Chapter 20: Telemetry & Analytics - **Not Started**
  - [ ] Chapter 21: Advanced Features - **Not Started**
- [ ] Part 11: Ecosystem & Future (Ch 26-28)
  - [ ] Chapter 26: Ecosystem & Tooling - **Not Started**
  - [ ] Chapter 27: Future Roadmap - **Not Started**
  - [ ] Chapter 28: Appendices - **Not Started**

**Status Key**: Not Started | In Progress | Complete | Needs Review

## Critical Planning Documents

Before beginning any documentation work, review these essential resources:

### Content Mapping (`/docs/rewrite/mapping.md`)
- **Purpose**: Maps archive files to the new structure with exact line numbers
- **Key Information**:
  - Location of each chapter in archive files
  - Line-by-line boundaries for sections 15-28
  - Discovery that spec.md is NOT the full 28-chapter version
  - Strategy for reconstructing chapters 1-14 from multiple sources
- **Use This For**: Finding specific content in archive files

### Implementation Plan (`/docs/rewrite/plan.md`)
- **Purpose**: Detailed roadmap for writing all 28 chapters
- **Key Information**:
  - Two-phase strategy (reconstruction vs extraction)
  - Per-chapter source identification and enhancement plans
  - 5-week implementation timeline
  - Quality assurance checklists
- **Use This For**: Understanding what to write and when

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

## General Prompt for New Agents

When starting a new session to continue this project, use this prompt:

```
I'm working on the GRAPHITE documentation rewrite project. Please read CLAUDE.md first to understand the project status and structure.

Current task: [Check the Chapter Progress Tracker in CLAUDE.md and work on the next uncompleted chapter]

Please:
1. Review the progress tracker to see what's been completed
2. Check the mapping.md file to find source content locations
3. Follow the plan.md guidelines for the chapter I'm working on
4. Update the progress tracker in CLAUDE.md when starting and completing work
5. Commit changes with descriptive messages following the project convention

The goal is to transform the 28-chapter technical specification into a well-organized 3-volume documentation set while maintaining all technical depth and adding helpful navigation/diagrams.
```

## How to Update Progress

When working on a chapter:
1. Update status to **In Progress** when starting
2. Update to **Complete** when finished
3. Update to **Needs Review** if you encounter issues or need clarification
4. Always commit CLAUDE.md updates so the next agent knows the current state

Example update:
```markdown
- [ ] Chapter 1: Introduction & Overview - **In Progress**
```
Changes to:
```markdown
- [x] Chapter 1: Introduction & Overview - **Complete**
```

---

*"Where every asset is a graph, and every graph tells a story."*

**Status: Ready to transform 28 chapters into organized volumes**