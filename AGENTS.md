# Repository Guidelines

## Project Structure

This repository is a collection of ordered study notes. Each topic has its own directory:

- `javaweb/` contains ten Chinese Markdown lessons covering frontend basics, Maven, HTTP, Spring Boot, databases, MyBatis, and employee-management practice.
- `linux/` and `redis/` contain the existing Linux and Redis notes.
- `python/` contains thirteen Chinese Python lessons plus a section index README, covering core syntax, object orientation, and four practice projects (AI, web scraping, data analysis, web development).
- `README.md` is the top-level learning index.

Keep source examples inside the lesson that explains them. Use relative links when linking between repository files.

## Editing and Validation

Notes are Markdown files encoded as UTF-8. Preserve the two-digit lesson prefix and the existing Chinese filenames, for example `javaweb/04-Web后端基础（基础知识）.md`. Use headings in numeric order, explain each new term before showing code, and end every lesson with a summary. Code fences should identify the language (`java`, `sql`, `xml`, `javascript`, or `bash`).

Before committing documentation changes, run these checks from the repository root:

```powershell
rg --glob '!python/*.md' '\*\*'
rg --glob 'javaweb/*.md' '^##? '
git diff --check
```

The first command excludes the `python/` lessons, which deliberately use `**bold**` to mark key terms, so the check stays quiet for newly added Python notes. Note that several existing Java-series notes also contain `**`; run the command without the `!python/*.md` exclusion if you want to see every current occurrence. The other checks catch heading and whitespace mistakes.

## Contributions

Use a short imperative commit subject with a clear scope, such as `docs: expand JavaWeb HTTP notes` or `docs: update learning index`. Keep unrelated topics in separate commits. Pull requests should describe which lessons changed, identify any new examples or diagrams, and include validation results. For formatting-only changes, screenshots are unnecessary; for rendered diagrams or layout changes, include a preview when useful.

## Safety

Do not commit passwords, database credentials, Baidu extraction codes beyond the public study links, generated build directories, or IDE metadata. Examples should use placeholders such as `${DB_PASSWORD}` and local test data.
