# CopilotCoworkUtilitySkills

This repository is a home for small, utility-focused Copilot Cowork skills. The goal is to build a practical collection of narrowly scoped skills that solve one task well, stay easy to maintain, and can be reused across broader Cowork workflows.

Each skill lives in its own folder with a `SKILL.md` file and is intended to be lightweight, focused, and independently iterable. Over time, this repo should grow into a toolbox of small helpers rather than a single large monolith.

Current contents:

- `redact-pdf/` - redacts specific words or phrases from PDFs by removing them from the text layer and covering them visually, while preserving other page text

- `signature-pdf/` - creates signature-ready PDFs that support AcroForm fields plus invisible Adobe Sign and DocuSign anchors

- `session-diagnostic/` - captures a detailed diagnostic log of a Cowork session, including routing, tool usage, subagent activity, metrics, and delivery verification
