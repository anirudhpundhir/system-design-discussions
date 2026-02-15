# CLAUDE.md - System Design Repository Guide

## Repository Purpose
This is a collaborative system design learning repository for interview preparation and career growth. It serves as a one-stop-shop for understanding, practicing, and mastering system design concepts.

## Repository Structure

```
system-design-discussions/
├── fundamentals/           # Core concepts (CAP, Design Principles, Patterns)
│   ├── cap-theorem/       # CAP, PACELC, consistency models
│   ├── design-principles/ # SOLID, scaling, distributed system principles
│   ├── scalability-patterns/ # Load balancing, caching, sharding
│   └── data-patterns/     # Storage, replication, partitioning
│
├── technologies/           # Technology Knowledge Base
│   ├── ai-ml/             # ML infrastructure, LLMs, vector DBs
│   ├── blockchain/        # Distributed ledger technologies
│   ├── distributed-systems/ # Consensus, clocks, coordination
│   ├── cloud-infrastructure/ # AWS, GCP, Azure, Kubernetes
│   ├── databases/         # SQL, NoSQL, NewSQL comparisons
│   ├── messaging-systems/ # Kafka, RabbitMQ, event streaming
│   ├── frameworks-languages/ # Tech stacks and languages
│   ├── low-latency/       # High-performance design patterns
│   ├── consistency-models/ # Strong, eventual, causal consistency
│   ├── legacy-systems/    # Modernization patterns
│   └── emerging-tech/     # Latest trends (edge, WASM, etc.)
│
├── design-problems/        # Individual system design problems
│   └── {problem-name}/
│       ├── README.md       # Problem statement and solution
│       ├── pre-reads/      # Prerequisites
│       ├── resources/      # External links, papers, videos
│       ├── diagrams/       # Mermaid/PlantUML/Miro diagrams
│       ├── code-snippets/  # Implementation examples
│       ├── notes/          # Key insights
│       ├── discussions/    # Trade-off analysis
│       └── feedback/       # Community improvements
│
├── templates/              # Reusable templates for contributions
├── resources/              # Global resources (learning paths, diagram tools)
├── weekly-discussions/     # Deep-dive session notes
└── contributors/           # Contributor profiles
```

## How to Use This Repository

### For Interview Preparation
1. Start with `fundamentals/` to build strong foundations
2. Follow the `INTERVIEW_APPROACH.md` template for structured problem-solving
3. Practice problems in `design-problems/` sorted by difficulty
4. Review `weekly-discussions/` for deep-dive insights

### For Contributing
1. Use templates in `templates/` directory
2. Follow the design problem structure
3. Add your analysis in `feedback/` sections
4. Participate in Sunday discussions

## Claude AI Integration Commands

### Understanding a Design
```
Explain the [design-name] system design, focusing on:
- Core requirements (functional and non-functional)
- High-level architecture
- Key components and their responsibilities
- Data flow and storage decisions
- Scalability considerations
```

### Comparing Approaches
```
Compare [approach-1] vs [approach-2] for [problem-name]:
- Trade-offs analysis
- When to use each
- CAP theorem implications
- Cost and complexity considerations
```

### Generating Diagrams
```
Create a Mermaid diagram for [component/system]:
- Show data flow
- Include all major components
- Highlight scaling points
- Mark potential bottlenecks
```

### Interview Simulation
```
Simulate a system design interview for [problem-name]:
- Ask clarifying questions
- Guide through the design process
- Point out improvements
- Suggest follow-up deep dives
```

### Deep Dive Topics
```
Explain [topic] in depth:
- How it works internally
- Implementation considerations
- Real-world examples
- Common pitfalls
```

## Design Problem Template Quick Reference

When tackling any design problem:
1. **Clarify Requirements** (2-3 min)
2. **Estimate Scale** (2-3 min)
3. **Define API** (3-5 min)
4. **High-Level Design** (10-15 min)
5. **Deep Dive Components** (10-15 min)
6. **Address Bottlenecks** (5-10 min)

## Key Principles

1. **No Single Best Design** - Every design has trade-offs
2. **Requirements Drive Decisions** - Business needs shape architecture
3. **Iterate and Improve** - Designs evolve with scale and requirements
4. **Document Rationale** - Why > What
5. **Learn from Feedback** - Community insights are valuable

## Diagram Tools Integration

This repo uses:
- **Mermaid** - Built into GitHub markdown (free, text-based)
- **Excalidraw** - Collaborative whiteboarding (free, JSON-based)
- **PlantUML** - UML diagrams (free, text-based)
- **tldraw** - Open-source whiteboard (free, can be self-hosted)

## Common Tasks for Claude

### Adding a New Design Problem
"Help me create a new design problem entry for [system-name] following the template structure"

### Reviewing a Design
"Review this design for [system-name] and suggest improvements considering CAP theorem and scalability"

### Creating Study Notes
"Summarize the key learnings from [design-problem] for quick revision"

### Preparing for Interview
"Create a 45-minute interview walkthrough for [design-problem] with expected follow-up questions"

### Technology Deep Dive
"Explain how [technology] works internally and when to use it vs alternatives"

### Creating Diagrams
"Create a Mermaid diagram showing [component/flow] for [system]"

### Trade-off Analysis
"Compare [option-A] vs [option-B] for [problem], covering performance, cost, complexity, and scalability"

### Learning Path Guidance
"What should I learn next after understanding [topic]? Create a study plan for [goal]"

### Mock Interview
"Conduct a mock system design interview for [system]. Ask me clarifying questions and guide me through the design"

### Doubt Resolution
"I don't understand [concept]. Explain it with examples and when it applies in real systems"

### Code Generation
"Generate a code snippet demonstrating [pattern/algorithm] in [language]"

### Real-World Examples
"How does [company] implement [feature/system]? What can we learn from their approach?"

## Code Conventions

- Diagrams: Use Mermaid for GitHub-native rendering
- Code snippets: Include language tags for syntax highlighting
- Documentation: Follow markdown best practices
- File naming: Use kebab-case (e.g., `rate-limiter-design.md`)

## Discussion Session Format

Regular deep-dive sessions cover:
1. A specific design problem
2. A fundamental concept
3. Real-world case studies
4. Q&A and doubt clearing

Notes are captured in `weekly-discussions/YYYY-MM-DD-topic.md`
