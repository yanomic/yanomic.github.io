---
name: blog-create
description: Creates a new Hugo blog draft in content/blogs using the repository naming convention. Use when the user asks to create a new blog post from a title or short description, and when date, categories, tags, or slug front matter must be prefilled.
---

# Blog Create

## Purpose

Create a new post as a Hugo leaf bundle at:

`content/blogs/YYYY-MM-DD-slug/index.md`

The input can be either:
- a clear title, or
- a short description that must be converted into a concise title first.

## Required defaults

Always prefill front matter with:
- `draft: true`
- `date` in `+0800` timezone, format: `YYYY-MM-DDTHH:mm:ss+0800`
- `categories` with at least one placeholder item
- `tags` with at least one placeholder item

## Workflow

1. Normalize the user input:
   - If input is a title, use it directly.
   - If input is a short description, derive a concise title in title case.
2. Build slug from the final title:
   - lowercase
   - words separated by `-`
   - remove punctuation
   - collapse repeated hyphens
3. Build directory name as `YYYY-MM-DD-slug` using current date in `+0800`.
4. Create a Hugo leaf bundle file:
   - path: `content/blogs/YYYY-MM-DD-slug/index.md`
5. If the directory already exists, append numeric suffix (`-2`, `-3`, ...).
6. Write front matter and a short content stub.

## Front matter template

Use this template exactly, replacing placeholders:

```yaml
---
draft: true
title: <Final Title>
slug: <slug>
date: <YYYY-MM-DDTHH:mm:ss+0800>
categories:
- General
tags:
- Blog
---

<!-- TODO: Write content -->
```

## Implementation notes

- Prefer Hugo-compatible structure (leaf bundle with `index.md`).
- Keep paths Unix-style, rooted at repository root.
- If title input is empty or too vague, ask for a better title/description before creating files.
