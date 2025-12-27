You are an expert in document drafting for technical documentation for software engineering. Your job is to build a prompt which I will then give to a coding agent to then iterively go through all of the listed files in the docs folder and construct all of the necessary documentation needed to drive a document driven development cycle of using coding agents to create code. Please instruct the LLM to draft the propmt to be given to the coding agent which will then only edit iterively all of the documents in the docs folder for our purpose.

Below is a complete blueprint explaining how to compose each file, what to include, and why each matters in a professional-grade software engineering process.

The goal is to create a living documentation ecosystem: every file works together, reducing ambiguity and aligning developers, designers, and AI collaborators.


Remember:
	•	Each .md file represents a single domain of truth.
	•	Documents should be interlinked (use relative Markdown links).
	•	Keep them modular — update one file without rewriting the others.
	•	Version control documentation changes like code — documentation is part of the codebase.

⸻

1. README.md — Project Overview and Orientation

Purpose: The entry point for humans and AI systems alike. It provides a high-level summary of what the project is, how to run it, and where to find everything.

Include:
	•	Project name and tagline
	•	Mission statement / goal
	•	System overview diagram
	•	Quick start guide (installation, setup, run)
	•	Directory structure (with descriptions)
	•	Tech stack (languages, frameworks, major dependencies)
	•	Links to all other major documents (architecture, standards, etc.)
	•	Contributor guide (how to fork, branch naming, PR etiquette)
	•	License information

⸻

2. requirements.md — Functional and Non-Functional Specs

Purpose: The contract between stakeholders and developers.

Include:
	•	Functional requirements: each user-facing feature, described in behavior-driven style (Given/When/Then).
	•	Non-functional requirements: performance, scalability, uptime, maintainability.
	•	Constraints: technology limits, APIs, third-party dependencies.
	•	Acceptance criteria per feature.
	•	Priority tags (P0 = critical, P1 = important, etc.)
	•	Future features (optional section for roadmap alignment).

⸻

3. architecture.md — System Design and Technical Blueprint

Purpose: Defines how the system is built, at both macro and micro levels.

Include:
	•	High-level architecture diagram (frontend, backend, DB, external services).
	•	Component breakdown: responsibilities, inputs/outputs, and dependencies.
	•	Data flow diagrams (DFDs or sequence diagrams).
	•	API design overview (link to OpenAPI/Swagger if applicable).
	•	Storage strategy: DB schema, indexes, caching, file storage.
	•	Error handling patterns.
	•	Scaling strategy (horizontal vs vertical).
	•	Change management process (how architecture evolves over time).

⸻

4. implementation.md — Development Details

Purpose: Provides actionable, developer-focused explanations on how architecture translates into code.

Include:
	•	Folder structure rationale.
	•	Class/module conventions (naming, organization).
	•	Dependency injection pattern.
	•	State management strategy.
	•	Configuration files overview.
	•	Environment variables list.
	•	Versioning approach (semantic versioning, changelog link).

⸻

5. standards.md — Coding, Naming, and Style Conventions

Purpose: Prevents chaos. Defines how code, commits, and documentation are written.

Include:
	•	Code style guide: indentation, naming, comments.
	•	Commit message convention (e.g., Conventional Commits).
	•	Branching strategy: main, develop, feature/*, etc.
	•	Documentation standards: how to write and update markdown.
	•	AI-generated code standards: prompts, review process, and approval.
	•	Linter/formatter settings reference (ESLint, Black, Prettier, etc.).
	•	CI/CD formatting enforcement policy.

⸻

6. sop.md — Standard Operating Procedures

Purpose: Describes the step-by-step workflow for recurring engineering tasks.

Include:
	•	Feature development workflow: ticket creation → branch → PR → review → merge → deploy.
	•	Code review checklist.
	•	Release management: version bump → tagging → changelog update.
	•	Incident response: error → alert → triage → resolution → postmortem.
	•	Backup & recovery procedures.
	•	Documentation update policy.
	•	AI usage SOP: when and how AI tools assist, and human verification required.

⸻

7. checklist.md — Quick Verification for Each Stage

Purpose: Provides a concise, repeatable quality assurance tool.

Include:
	•	Pre-commit checklist (lint, tests, docs updated).
	•	Pre-PR checklist (code reviewed, screenshots added).
	•	Pre-deploy checklist (env verified, rollback ready).
	•	Accessibility and SEO verification.
	•	Security scan checklist.
	•	Post-deploy validation checklist.

⸻

8. testing.md — Quality Assurance Strategy

Purpose: Defines the philosophy and specifics behind automated and manual testing.

Include:
	•	Testing strategy overview (unit, integration, E2E, load).
	•	Test pyramid diagram.
	•	Frameworks and tools (pytest, Jest, Cypress, etc.).
	•	Mocking and test data policy.
	•	Coverage standards (e.g., 80% minimum).
	•	Continuous testing setup.
	•	Bug reporting workflow and template.
	•	AI regression testing process (if applicable).

⸻

9. deployment.md — DevOps and Environment Strategy

Purpose: Explains how code moves from dev → staging → production.

Include:
	•	Environments overview (dev/staging/prod).
	•	CI/CD pipeline steps.
	•	Infrastructure diagram (servers, containers, cloud resources).
	•	Secrets management policy.
	•	Rollback plan.
	•	Monitoring & logging strategy.
	•	Performance benchmarks.
	•	Post-deployment verification.

⸻

10. security.md — Secure Development Lifecycle (SDLC)

Purpose: Ensures security is embedded, not bolted on.

Include:
	•	Threat model overview.
	•	Authentication & authorization design.
	•	Encryption practices (data at rest/in transit).
	•	API security best practices.
	•	Dependency vulnerability scanning process.
	•	Access control policies.
	•	Incident response protocol.
	•	Penetration testing schedule.

⸻

11. accessibility.md — Inclusive and Compliant Design

Purpose: Guarantees usability across ability levels.

Include:
	•	Accessibility philosophy (e.g., “Accessible by design”).
	•	Compliance standards: WCAG 2.1, ADA, Section 508.
	•	Color contrast, typography, and ARIA guidelines.
	•	Keyboard navigation checklist.
	•	Screen reader testing workflow.
	•	Accessibility audit frequency.
	•	Tool references: Lighthouse, axe-core, etc.

⸻

12. seo.md — Search Engine Optimization Blueprint

Purpose: For web-facing software, ensures visibility and ranking.

Include:
	•	Keyword strategy (linked to product goals).
	•	Content metadata rules: title, description, alt text.
	•	Schema markup usage.
	•	Internal linking standards.
	•	Performance & mobile SEO optimization.
	•	Sitemaps and robots.txt policy.
	•	AI-generated content review checklist (avoid keyword stuffing, ensure readability).

⸻

13. ai_guidelines.md — Responsible and Effective AI Integration

Purpose: Defines how AI is used ethically and consistently in the codebase or workflow.

Include:
	•	AI usage principles (transparency, verification, attribution).
	•	Prompt engineering standards.
	•	Evaluation and bias mitigation process.
	•	Data privacy considerations.
	•	Model selection and retraining strategy.
	•	Human-in-the-loop checkpoints.
	•	Audit logs for AI-generated output.

⸻

14. system_prompt.md — The Canonical Prompt for AI Agents

Purpose: The “personality” and rules of your project’s AI assistants (local or remote).

Include:
	•	Project identity and mission.
	•	Role definition (what the AI should and shouldn’t do).
	•	Tone, formatting, and output style.
	•	Context injection rules (which docs or repos to reference).
	•	Safety and refusal guidelines.
	•	Update history of prompt iterations.