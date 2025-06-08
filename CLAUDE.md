# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Lint Commands
- Install dependencies: `npm install`
- Run textlint: `npx textlint articles/*.md`
- Preview blog locally: `npx zenn-cli preview`
- Create new article: `npx zenn-cli new:article`

## Content Guidelines
- Article frontmatter format: title, emoji, type, topics, published
- Use Zenn's markdown format for articles
- Follow Japanese technical writing style guidelines:
  - Use "ですます" style in body text
  - Add spaces between half-width and full-width characters
  - Surround inline code with spaces
  - End sentences with "。" or ":" punctuation
  - Allowed to use full-width exclamation/question marks
  - Multiple occurrences of particle "か" are permitted
  - No specific character limit on sentences

## File Structure
- Place articles in `/articles` directory
- Use descriptive filenames for articles
- Images should be placed in `/images` directory