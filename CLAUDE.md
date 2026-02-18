# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository is a Claude Code plugin marketplace for [Gerrit](https://www.gerritcodereview.com/), the open-source code review tool.

## Repository Structure

- `.claude-plugin/marketplace.json` — Marketplace manifest listing available plugins
- `plugins/gerrit/` — Gerrit plugin directory
- `plugins/gerrit/.claude-plugin/plugin.json` — Plugin manifest
- `plugins/gerrit/skills/gerrit/SKILL.md` — Gerrit skill definition
- Licensed under Apache 2.0

## Git Workflow

This project uses GerritHub for code review. The remote is accessed via SSH:
```
ssh://DanieleSassoli@gerrithub.io:29418/DanieleSassoli/gerrit-skills
```

A Gerrit `commit-msg` hook is installed to generate Change-Id footers. When committing, ensure the Change-Id is present in commit messages.

To push changes for review:
```bash
git push origin HEAD:refs/for/main
```
