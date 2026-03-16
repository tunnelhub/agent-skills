# TunnelHub Agent Skills

Public agent skills for the TunnelHub ecosystem.

## Available Skills

- `tunnelhub-sdk` - guidance for building and maintaining integrations with `@tunnelhub/sdk`

## Install With skills.sh

Install the TunnelHub SDK skill for OpenCode:

```bash
npx skills add tunnelhub/agent-skills --skill tunnelhub-sdk -a opencode
```

Install globally:

```bash
npx skills add tunnelhub/agent-skills --skill tunnelhub-sdk -a opencode -g
```

List skills in this repository:

```bash
npx skills add tunnelhub/agent-skills --list
```

## Repository Structure

```text
skills/
  tunnelhub-sdk/
    SKILL.md
    references/
```

## Local Development

You can validate the repository locally before publishing it:

```bash
npx skills add ./ --list
```
