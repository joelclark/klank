# VM Agent Harness: Requirements (Draft for Review)

## 1. Purpose

Run several Claude Code efforts at the same time, each in its own clean Linux VM. Every VM is a disposable copy of a "master" VM that looks and behaves like my own dev environment: same user, dotfiles, `gh` login and Claude login. The agent works there the way I would. It branches, commits, pushes, opens PRs, reads review feedback and responds to it.

The goal is **parallelism with isolation**. Separate working trees, builds and test runs mean efforts never collide.

How these requirements are met (languages, tools, structure) is left to the implementer.

## 2. Scope

### In scope (v1)
- A single Linux host with KVM.
- One master VM image, versioned and refreshable.
- Worker VMs created as copy-on-write clones of a specific master version.
- Driving Claude Code inside workers non-interactively from the host, plus manual interactive access.
- A PR feedback loop that feeds new review comments back into the right session.
- Lifecycle management: spawn, suspend, resume, reap.

### Out of scope (v1)
- Cloning a *running* master (memory snapshots). Workers boot fresh from disk.
- Multiple hosts or a cluster scheduler.
- macOS or Windows guests.
- Credential isolation between workers and my account. This is an accepted risk; see §6.
- A web UI.

## 3. Terms

| Term | Meaning |
|---|---|
| **Master** | A versioned base disk image containing my user, tools, dotfiles and pre-cloned repos. Never modified once created. |
| **Current master** | The master version that new workers are built from. |
| **Worker** | A VM booted from a copy-on-write clone of one specific master version. |
| **Effort** | One unit of work, usually one GitHub issue or one thread. One effort maps to one worker. |
| **Turn** | One non-interactive Claude Code run inside a worker. |
| **Session** | The Claude Code conversation for an effort, continued across turns. |

## 4. Functional Requirements

### 4.1 Master image management

- **FR-1** The master must have a user whose username and UID match my host account.
- **FR-2** The master must contain:
  - Claude Code, `gh`, `git`, and the project toolchains.
  - A clone of each target repo.
  - git configured so that pushes use my `gh` login without prompting.
- **FR-3** Refreshing the master must produce a **new** master version. It must never modify an existing version in place. A refresh must:
  1. Bring in my current dotfiles and config from the host, from a configurable list. The defaults are:
     - `~/.gitconfig`
     - `~/.config/gh`
     - `~/.claude/settings.json`
     - `~/.claude/CLAUDE.md`
     - skills, agents, MCP config and shell rc as listed
  2. Ensure correct ownership of my home directory.
  3. Update each repo to the latest default branch, as my user.
  4. Run optional per-repo prep steps, such as installing dependencies.
  5. Make the new version current atomically.
- **FR-4** A failed refresh must leave the current master untouched and clean up any partial version.
- **FR-5** A master version must not be deleted while any worker depends on it. Unreferenced versions must be prunable.

### 4.2 Worker lifecycle

- **FR-6** Spawning a worker must clone a **concrete** master version, never a moving "current" pointer, then boot it and wait until it is reachable over SSH.
- **FR-7** Workers must be reachable from the host as `ssh <worker-name>`.
- **FR-8** Each worker must get its own hostname, MAC address and SSH host keys. A per-worker machine-id is preferred.
- **FR-9** On first boot, a worker must fetch the latest changes in its repos before the first turn. A stale master must only cost time, never produce wrong code.
- **FR-10** Idle workers must be suspendable to disk and resumable on demand. Sending a turn to a suspended worker must resume it automatically.
- **FR-11** A worker must be torn down, with its disk clone deleted, when any of these happens:
  - its PR is merged or closed
  - it has been idle longer than a configurable timeout
  - I tear it down manually
- **FR-12** Per-worker vCPU, RAM and maximum disk size must be configurable, with a global default.

### 4.3 Identity and authentication

- **FR-13** Claude Code in workers must authenticate with a long-lived subscription token from `claude setup-token`, supplied via the `CLAUDE_CODE_OAUTH_TOKEN` environment variable.
- **FR-14** My interactive Claude login (`~/.claude/.credentials.json`) must **not** be copied into the master. Many VMs refreshing one shared login is avoided by design.
- **FR-15** Claude Code in workers must load my normal configuration: settings, CLAUDE.md, skills and MCP servers.
  - Note: Claude Code's "bare" mode skips this configuration and does not read `CLAUDE_CODE_OAUTH_TOKEN`, so it is not suitable here.
- **FR-16** `gh` and git push must work in workers with my identity, with no prompts.

### 4.4 Driving Claude Code

- **FR-17** The harness must run turns non-interactively, in the worker's repo directory, with the prompt supplied by the harness.
- **FR-18** Turns must never block waiting for a permission approval. Anything that would prompt is denied, and the denial is recorded.
- **FR-19** The first turn of an effort must establish a session. Every later turn for that effort must continue the same session.
- **FR-20** The full structured output of every turn must be saved on the host, per effort and per turn. The final outcome must be stored alongside it, including result text, exit status, cost and any permission denials.
- **FR-21** Only one turn may run per worker at a time. A second turn for a busy worker must queue or fail clearly.
- **FR-22** I must be able to SSH into any worker and continue its session interactively in Claude Code. While I'm attached, the harness must not start turns on that worker.
- **FR-23** A turn exceeding a configurable wall-clock limit must be interrupted gracefully, so the session stays resumable, before any forced kill.

### 4.5 Effort intake

- **FR-24** I must be able to start an effort from a GitHub issue reference. The harness:
  1. Spawns a worker.
  2. Builds the first prompt from the issue title and body plus a configurable instruction template: branch naming, open a draft PR, link the issue.
  3. Runs the first turn.
- **FR-25** I must be able to start an effort from a free-form prompt, for non-issue threads.
- **FR-26** The harness must keep a persistent registry of efforts: name, repo, issue, PR number, session identifier, master version, state and last activity.

### 4.6 PR feedback loop

- **FR-27** The harness must watch each active effort's PR for review comments, review summaries and conversation comments newer than the effort's last turn.
- **FR-28** When new feedback appears, the harness must continue that effort's session with a prompt saying there is new feedback on PR #n to read and address.
- **FR-29** The harness must not react to comments the agent itself posted, to avoid loops. Since the agent posts as me, this needs a defined rule; see Open Questions.
- **FR-30** When a PR is merged or closed, the harness must mark the effort done and trigger teardown.

### 4.7 Visibility

- **FR-31** I must be able to list all efforts with:
  - state (running / idle / suspended / done)
  - PR link
  - last turn time and outcome
  - cumulative cost
- **FR-32** I must be able to view a readable rendering of an effort's latest turn: assistant text and tool calls.

## 5. Non-Functional Requirements

- **NFR-1 Concurrency:** support at least *N* simultaneous workers on one host. *N* is TBD.
- **NFR-2 Spawn time:** a new worker should be SSH-ready within ~30 s. Target TBD.
- **NFR-3 Disk:** a worker's disk clone should start near zero size and grow only with the effort's changes, up to a configurable maximum.
- **NFR-4 Master safety:** nothing may write to a master version that has dependent workers.
- **NFR-5 Idempotence:** every operation must be safe to retry after a failure, without corrupting state or leaking VMs.

## 6. Accepted Risks

- **Full credentials in every worker.** Each worker can push to any repo and act as me on GitHub. This is accepted for v1.
  - Possible mitigation: use a fine-grained GitHub token limited to the target repos.
- **Prompt injection via issue and PR text.** An agent reading untrusted text runs with my credentials and without permission prompts.
  - Possible mitigation: only start efforts from issues I've labeled, and only feed back comments from allowed authors.
- **No egress filtering in v1.** Workers have the same network access as my machine.

## 7. Open Questions

1. **Loop prevention:** the agent posts as me, so how does the harness tell my real feedback from the agent's own replies? Options:
   - a hidden marker in agent comments
   - only react to formal review submissions
   - only react to specific commenters
2. **Trigger model:** manual start only, or also auto-start when an issue gets a specific label?
3. **Session continuity after teardown:** if a worker is torn down while its PR is still open (idle timeout), should a later comment respawn it and restore the session and branch? Or should the effort simply be closed?
4. **Permission policy:** should workers allow broad autonomous action, or a more restricted, explicit set of allowed tools and commands?
5. **Concurrency target (*N*)** and per-worker resource defaults.
6. **Multiple repos:** one master with all repos, or one master per repo?
7. **Cost guardrails:** should there be a per-turn, per-effort or per-day spend cap?
8. **Master refresh cadence:** manual only, or scheduled (e.g., nightly)?

## 8. Acceptance Criteria (v1)

- [ ] Refreshing the master produces a new version and leaves existing workers unaffected.
- [ ] Three efforts can run at the same time on separate issues, each producing its own draft PR, with no interference.
- [ ] A review comment on one PR causes only that effort's session to continue and push a follow-up commit.
- [ ] I can SSH into any worker and continue its session interactively.
- [ ] Merging a PR leads to its worker being torn down. Unreferenced master versions are prunable.
- [ ] I can see the state, PR and cumulative cost of every effort at a glance.
