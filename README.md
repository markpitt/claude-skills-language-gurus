<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/banner-dark.svg">
    <img src="docs/assets/banner-light.svg" alt="Language gurus" width="100%">
  </picture>
</h1>

Current best practice for Python, TypeScript, Rust, Go, shell and PowerShell.

One family with one shape: each skill enforces current practice for its language. `dotnet-guru` is the seventh member and lives in [claude-skills-dotnet](https://github.com/markpitt/claude-skills-dotnet).

This is one of seven skill groups. [claude-skills](https://markpitt.dev/claude-skills/) lists them all and
holds the shared tooling: validation, packaging, the evaluation harness and the TUI.

## Skills

<!-- BEGIN GENERATED SKILL INDEX -->
| Skill | Bundled files | What it does |
|---|---|---|
| [golang-guru](skills/golang-guru/SKILL.md) | 6 files | Enforces Go 1.27 best practices when writing, reviewing, or refactoring Go code. |
| [pwsh-guru](skills/pwsh-guru/SKILL.md) | 6 files | Enforces PowerShell 7.6 best practices when writing, reviewing, or refactoring PowerShell scripts and modules. |
| [python-guru](skills/python-guru/SKILL.md) | 6 files | Enforces Python 3.12+ best practices when writing, reviewing, or refactoring Python code. |
| [rust-guru](skills/rust-guru/SKILL.md) | 8 files | Enforces current Rust best practices when writing, reviewing, or refactoring Rust code. |
| [shell-guru](skills/shell-guru/SKILL.md) | 5 files | Enforces current shell scripting best practices when writing, reviewing, or refactoring Bash/POSIX sh scripts. |
| [typescript-guru](skills/typescript-guru/SKILL.md) | 6 files | Enforces TypeScript 7 best practices when writing, reviewing, or refactoring TypeScript code. |

6 skills.
<!-- END GENERATED SKILL INDEX -->

## Install

Copy a skill directory into either skills location, then start a new session:

```bash
cp -r skills/<skill-name> ~/.claude/skills/    # personal, all projects
cp -r skills/<skill-name> .claude/skills/      # this project only
```

For Claude Desktop, zip the skill directory's contents and upload the zip under Settings,
Capabilities, Skills:

```bash
cd skills/<skill-name> && zip -r ../../<skill-name>.zip .
```

## Licence

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Each skill directory carries its
own copy of the licence, so a skill copied out on its own stays covered.
