# Repository Guidelines

## Project Structure

This repository is a collection of ordered study notes. Each topic has its own directory:

- `java/` contains the Java core lessons (`00` 学习总纲 plus `01`-`09`), covering syntax, object orientation, collections, GUI and Stream, exceptions and IO, multithreading and networking, reflection and dynamic proxies.
- `javaweb/` contains the JavaWeb lessons covering frontend basics, Maven, HTTP, Spring Boot, databases, MyBatis, and employee-management practice. The directory holds 20 numbered topics plus `README.md`; the 14th topic is split into `14A`/`14B` files, so the file count is larger than the topic count.
- `mysql/` contains the MySQL lessons from installation and SQL through transactions, indexes, locks, logs, replication, sharding, and read-write splitting.
- `redis/` contains the Redis lessons, from basic types through the practical cases (login, seckill, feed, GEO) to persistence, clustering, expiry, best practices, internals, networking, and multi-level caching.
- `linux/` contains the Linux lessons, from concepts and virtual machines through shell commands, users, permissions, practical operations, shell scripting, text processing, disks, SSH, troubleshooting, and service deployment.
- `python/` contains thirteen Chinese Python lessons plus a section index README, covering core syntax, object orientation, and four practice projects (AI, web scraping, data analysis, web development).
- `README.md` is the top-level learning index.

Keep source examples inside the lesson that explains them. Use relative links when linking between repository files.

## Editing and Validation

Notes are Markdown files encoded as UTF-8. Preserve the two-digit lesson prefix and the existing Chinese filenames, for example `javaweb/04-Web后端基础（基础知识）.md`. Use headings in numeric order, explain each new term before showing code, and end every lesson with a summary. Code fences should identify the language (`java`, `sql`, `xml`, `yaml`, `properties`, `javascript`, `bash`, `text`, or `mermaid`).

New or expanded lessons follow the house style: an `#` title, a short introduction, `## 1. 学习目标与前置知识`, concept-first sections with comparison tables and Mermaid diagrams, then `## N. 小白易错点`, `## N. 练习清单`, and `## N. 资料对应关系` (README links should point at real files). Use tables for横向 and 纵向 comparisons, and Mermaid (`flowchart`, `sequenceDiagram`) for flows and data paths. Do not use `**bold**` or `---` horizontal rules in the Java-series notes; the Python lessons deliberately keep their own bold convention.

Before committing documentation changes, run these checks from the repository root.

With ripgrep installed:

```bash
rg --glob '!python/*.md' '\*\*'
rg --glob 'javaweb/*.md' '^##? '
git diff --check
```

Without ripgrep, the equivalent PowerShell checks are:

```powershell
Get-ChildItem -Recurse -Filter *.md | Where-Object { $_.DirectoryName -notmatch 'python$' } |
    Select-String -Pattern '\*\*'
Get-ChildItem javaweb\*.md | Select-String -Pattern '^##? '
Get-ChildItem -Recurse -Filter *.md | Select-String -Pattern '[ \t]+$'
git diff --check
```

The first check excludes the `python/` lessons, which deliberately use `**bold**` to mark key terms. A few remaining `**` hits outside `python/` are legitimate: `/** */` javadoc in `java/01`, `xxxValue()` method names, and the C pointer declaration `dictEntry **table` in `redis/14`. The other checks catch heading and whitespace mistakes. When adding a lesson, also confirm it has a single `#` heading, no placeholder markers such as `@@APPEND@@` or `<!-- 继续 -->`, and no code fence without a language tag.

## Contributions

Use a short imperative commit subject with a clear scope, such as `docs: expand JavaWeb HTTP notes` or `docs: update learning index`. Keep unrelated topics in separate commits, and prefer one commit per topic when a batch of edits spans several directories. Pull requests should describe which lessons changed, identify any new examples or diagrams, and include validation results. For formatting-only changes, screenshots are unnecessary; for rendered diagrams or layout changes, include a preview when useful.

## Safety

Do not commit passwords, database credentials, cloud access keys, Baidu extraction codes beyond the public study links, generated build directories, or IDE metadata. Examples should use placeholders such as `${DB_PASSWORD}`, `${OSS_ACCESS_KEY_ID}`, and `${SERVER_IP}` together with local test data.
