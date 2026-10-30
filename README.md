# Carats Skills

Agent skills for building apps with [Carats](https://github.com/carats-labs), a full-stack framework similar to Next.js that uses JJSX instead of React.

These skills teach AI coding agents (Claude Code, Cursor, Codex, Windsurf, and others) how to write idiomatic Carats code: routes, components, culets, hooks, and styling.

## Install

Install every skill in this repository:

```bash
npx skills add carats-labs/skills --all
```

Pick skills interactively:

```bash
npx skills add carats-labs/skills
```

Install globally instead of per-project:

```bash
npx skills add carats-labs/skills --all -g
```

Install for a specific agent:

```bash
npx skills add carats-labs/skills --all --agent claude-code
```

List what is available without installing:

```bash
npx skills add carats-labs/skills --list
```

> **Install the whole bundle.** The skills reference each other
> (for example, `carats-client` requires `carats-essentials`),
> so installing a single skill may leave the agent with missing context.

## Skills

| Skill | Use it for |
| --- | --- |
| `carats-essentials` | Core concepts, project setup, tsconfig, scripts. Read first. |
| `carats-client` | Client-side pages, components, facets, routing, `Burnish`. |
| `carats-server` | Culets, server entrypoint, server-side rendering and props. |
| `carats-hooks` | TODO: one-line description |
| `carats-styling` | TODO: one-line description |
| `carats-detailed-types` | TODO: one-line description |

## Update

```bash
npx skills update
```

## Project template

A complete working app that follows these skills:

```bash
bun create carats <directory>
```