---
name: workspace
description: Find what the target's runs need besides their own clone (other repositories, data, buckets, services, credentials) and declare it at item scope with declare_workspace_input, credentials named and never held; missing credentials and their console links go to the plan summary. Never request_access.
---

# The workspace

A task's **workspace** is what its runs receive besides their clone: a mount (a bucket prefix,
live), a folder copied from a connector (another repository at a ref, a bucket prefix, an HTTPS
link), a file or folder a person uploads, and the credentials those read with. You declare it on
the item, so only that item's runs receive it: a phase's run receives its own item's inputs and
every container's above it.

## 1. What is there

- Your working directory is the target's repository; every other repository of the project is a
  read-only copy under `~/repos/<owner>/<repo>` at its default branch. Those copies are yours: a
  coding run receives only what is declared for it.
- "Already in your sandbox" in your instructions lists the target's mounts and folders, with
  their sources and targets.
- `list_workspace(item_id=<item>)`: what a coding run of the item would receive, from its own
  scope upward, mounts first, each with `connector`, `source`, `target`, `required` and its
  `credential.status`. `list_inputs(project_id=<project>)`: every input's name, kind, class and
  scope, never a value. `list_repositories`: the project's repositories and default branches.

## 2. What the work needs

Read the Spec's **Workspace** section first: it is the person's list. Then read the code: imports
from a sibling repository, paths the code reads that the repository does not hold, variables it
reads that nothing sets, services it calls, data the acceptance criteria name. For each need, find
its source (which repository and ref, which bucket and prefix, which URL) and whether a run already
receives it. Something you cannot source is an open question, never a guess.

## 3. Declare

Declare on the item whose runs need it: the feature when several of its phases do, a phase when
only it does, the target itself for a task or a bug.

```
declare_workspace_input(
  item_id=<the item>, name=<unique, short>, kind="mount" | "folder" | "file",
  connector="s3" | "gcs" | "azure-blob" | "git" | "https",   # none for a person's upload
  source="<bucket>/<prefix>" | "owner/repo" | <URL>,
  options={"ref": "<branch, tag or commit>"} | {"endpoint": ..., "region": ...},
  target="~/data/ref" | "~/repos/<owner>/<repo>" | <a path in the clone>,
  access="read-only",          # "read-write" with writes="per-run" only when the work writes
  credential=<a credential input's name>,   # omitted: a public source, an upload, a linked repository
  description=<what the work uses it for, in one line>)
```

- Item scope only. Never `repository_id` or `project_id` scope, never `profiles` with `gate`,
  never `writes="exclusive"` (`PLANNER_SCOPE`). What you declare is agent-authored: coding runs
  receive it, the gate only once a person endorses it.
- A Git folder of a repository the project links needs no credential (`not_needed`): use the
  repository's default branch as `ref` unless the Spec names another.
- A credential is named, never held: reuse the name of an existing credential input
  (`list_inputs`) that can read the source; else choose a descriptive name. Never ask anyone for a
  value, never put a value, token or signed link in any argument, comment or summary.

## 4. The reply

- `credential.status` `present` or `not_needed`: declared. Note the `input_id`, the target and the
  `presence`.
- `missing`: nothing was declared. Record the row, `credential.name` (the name to declare it with
  next time), what the credential must be able to do (read that bucket, say) and
  `credential.console_url` (the console's form, pre-filled, where a person enters it). They go in
  the summary's **Missing credentials**; the person enters them and sends the plan back, and your
  next pass declares the row with `credential=<credential.name>`. Do not ask a question for it, and
  never call `request_access`: that is a coding agent's tool, not yours.
- A refusal: `MOUNT_UNSUPPORTED` (the cluster cannot mount that connector: declare a folder instead
  when a copy at run start is enough, else record it), `INVALID_CREDENTIAL` (its message names the
  scopes), `CONNECTOR_KIND`, `PLANNER_SCOPE`: record the code and the message for the summary; do
  not retry the same call.
- A service (an MCP server, a REST API, a database) is not declared with this tool: list it in the
  summary for a person to add in the console.

## 5. Name it in the plan

Every phase or task that uses a declared row names it: in a phase plan's `### Requires` as
`workspace: <name> at <target> (<kind>, <access>) — feature workspace`, and in "What to build" what
it is for, so the implementing agent finds it.

## 6. Artifacts

A document the plan produces for the implementer (a research note, a schema sketch, a sample) may
be attached to an item within your reach: `add_item_input(item_id, name, kind="file", target)`
then `zc-input put <input id> <path>` in your shell. Attach only what the plan relies on.
