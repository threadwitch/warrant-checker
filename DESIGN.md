# warrant Frontmatter Validation Checker

## What It Does
This is intended to be a frontmatter schema builder and checker for Markdown files.  The files in a repo for a project, including, but not limited to a readme, a design doc, a process log, and a decision record.  This could also include skills, or specific instructions in the project's folders. The goal is to systematize and check for a given set of files, and allow agents to pull them into memory as needed.

## Who It's For
It's for myself at first, but the idea is to generalize it to work for any given repo, to allow for custom frontmatter schemas, even on a per-project basis.  It's designed to make human-readable and editable files easier for an agent to call programmatically with hooks in a harness with support for it.  This is to enable human collaboration with agents in a structured way.

## Build Order
- __Layer 0:__ Before anything else, we need to tighten up the spec docs, eliminate the redundacies, and have the shape that we're building to clearly laid out.
- __Layer 1:__ The first step is a smaller, more focused CLI, check a project for a design doc, a process file, readme, and some skills.  The warrant output can be passed to stdout here.
- __Layer 2:__ Add the watcher daemon to the process.
- __Layer 3:__ Adding the decision tracking portion of the spec and start dogfooding.

## Tech Choices
- KDL for the frontmatter. It gives expressiveness and depth, while enforcing correctness.
- Rust for the tooling.  The usual reasons.
