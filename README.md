# @pokujs/skills

This repository contains agent-facing documentation for working with Poku.

The main entry point is [SKILL.md](./SKILL.md), which describes when to use the skill and provides a concise guide to Poku's testing model. The supporting Markdown files expand on the areas that usually need deeper reference material.

## Contents

- [SKILL.md](./SKILL.md): main skill definition and quick-reference guide
- [assertions.md](./assertions.md): assertion APIs and examples
- [background-services.md](./background-services.md): background processes, ports, and readiness helpers
- [configuration.md](./configuration.md): configuration options, reporters, and CLI flags
- [patterns.md](./patterns.md): unit, integration, and end-to-end testing patterns

## Purpose

Poku is a cross-runtime test runner for Node.js, Bun, and Deno. This repository packages the guidance needed for agents and contributors to answer questions, write tests, and debug test suites accurately.

## Usage

Use this repository as reference material when:

- writing or debugging tests with Poku
- explaining how Poku executes test files
- looking up assertion APIs or runtime-specific behavior
- working with background services in integration or e2e tests
- checking configuration and CLI options

## Maintenance

When updating the skill, keep `SKILL.md` aligned with the supporting reference files so the overview and the deeper documentation do not drift apart.