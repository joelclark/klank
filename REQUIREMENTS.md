# VM Agent Harness: Requirements (Draft for Review)

## 1. Purpose

Run several Claude Code efforts at the same time, each in its own clean Linux VM. Every VM is a disposable copy of a "master" VM that looks and behaves like my own dev environment: same user, dotfiles, `gh` login and Claude login. The agent works there the way I would. It branches, commits, pushes, opens PRs, reads review feedback and responds to it.

The goal is **parallelism with isolation**. Separate working trees, builds and test runs mean efforts never collide.

How these requirements are met (languages, tools, structure) is left to the implementer, within the constraints in §2.1 and §5.

### 1.1 Premises

- **Full VMs are a deliberate choice**, not a conclusion derived from a comparison with worktrees or containers. Planners should not re-litigate it.
- **Trusted inputs for v1.** Efforts run against private repos, and every author of issues, PRs and comments is trusted. This is what makes the v1 security posture in §6 acceptable. The design must leave room to tighten it later (NFR-6).
- **A native desktop command center** (§4.8) is part of the product. It is detailed later, but it affects architecture now (§2.1).

## 2. Scope

### In scope (v1)
- A single Linux host with KVM. Supported host distributions: **Omarchy** (Arch-based, Wayland/Hyprland) and **Fedora**.
- One master VM image, versioned and refreshable.
- Worker VMs created as copy-on-write clones of a specific master version.
- Driving Claude Code inside workers non-interactively from the host, plus manual interactive access.
- A PR feedback loop that feeds new review comments back into the right session.
- Lifecycle management: spawn, suspend, resume, reap.
- A native desktop **command center** for managing efforts (§4.8). Its detailed design is deferred, but its existence is a v1 architectural input.

### 2.1 Architectural constraints

These follow from the command center and the host platforms. Planners must account for them even where the command center itself is not yet specified.

- The harness's capabilities must be available through **one programmatic interface** that both the CLI and the command center use. No capability may exist only inside a client.
- That interface must support **live updates** (push or subscription), not only polling, so a GUI can reflect state changes as they happen.
- The harness must keep running independently of any client. Closing the command center must not stop, or change the state of, any effort.
- Machine-readable state (registry, turn records, events) is the source of truth. Anything a client displays must be derivable from it.

### Out of scope (v1)
- Cloning a *running* master (memory snapshots). Workers boot fresh from disk.
- Multiple hosts or a cluster scheduler.
- macOS or Windows guests.
- Credential isolation between workers and my account. This is an accepted risk; see §6.
- A web UI, and any Electron-based client.
- Input scrubbing and egress filtering. Deferred, but a seam is required (NFR-6).

## 3. Terms

| Term | Meaning |
|---|---|
| **Master** | A versioned base disk image containing my user, tools, dotfiles and pre-cloned repos. Never modified once created. |
| **Current master** | The master version that new workers are built from. |
| **Worker** | A VM booted from a copy-on-write clone of one specific master version. |
| **Effort** | One unit of work, usually one GitHub issue or one thread. One effort maps to one worker. |
| **Turn** | One non-interactive Claude Code run inside a worker. |
| **Session** | The Claude Code conversation for an effort, continued across turns. |
| **Command center** | The native desktop application I use to monitor and manage efforts. See §4.8. |

## 4. Functional Requirements

### 4.1 Master image management

- **FR-1** The master must have a user with the same username as my host account, so my home directory has the same path on the host and in every worker.
  - Absolute paths in the configuration I bring over (settings, hooks, MCP config, `CLAUDE.md`) and in per-project data must resolve identically in a worker.
  - Every file brought in from the host must end up owned by this user, with correct permissions. A matching UID is not required, but ownership must be set explicitly (FR-3, step 2).
- **FR-2** The master must contain:
  - Claude Code, pinned to a specific version with auto-update disabled, `gh`, `git`, and the project toolchains.
  - A clone of each target repo.
  - git configured so that pushes use my `gh` login without prompting.
  - Everything each repo needs to build and test, defined per repo: private package registry credentials, env files, and local services such as databases or Docker. Anything secret here falls under the credential risk in §6.
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
- **FR-10** Workers that have been idle for a configurable period must be suspended, keeping their disk (and so their session) intact, and resumable on demand. Sending a turn to a suspended worker must resume it automatically.
- **FR-11** A worker must be torn down, with its disk clone deleted, when any of these happens:
  - its PR is merged or closed
  - it has been suspended longer than a separate, much longer configurable cutoff
  - I tear it down manually
- **FR-39** If new feedback arrives on an open PR whose worker was torn down for age or by hand, the harness must **revive** the effort: spawn a new worker from the current master, check out the PR's remote branch, and start a fresh session seeded with the PR context. Restoring the old session is not required. Before any teardown, the harness must copy the effort's session transcript to the host.
- **FR-12** Per-worker vCPU, RAM and maximum disk size must be configurable, with a global default.

### 4.3 Identity and authentication

- **FR-13** Claude Code in workers must authenticate with a long-lived subscription token from `claude setup-token`, supplied via the `CLAUDE_CODE_OAUTH_TOKEN` environment variable.
- **FR-14** My interactive Claude login (`~/.claude/.credentials.json`) must **not** be copied into the master. Many VMs refreshing one shared login is avoided by design.
- **FR-15** Claude Code in workers must load my normal configuration: settings, CLAUDE.md, skills and MCP servers.
  - Note: Claude Code's "bare" mode skips this configuration and does not read `CLAUDE_CODE_OAUTH_TOKEN`, so it is not suitable here.
  - Headless operation must not be broken by first-run onboarding or workspace-trust prompts. Every master version must pass a smoke test before it becomes current: in a throwaway clone, run a trivial headless turn and confirm that settings, skills and MCP servers loaded, that no interactive prompt appeared, and that `gh` is authenticated. A failed smoke test fails the refresh (FR-4).
- **FR-16** `gh` and git push must work in workers with my identity, with no prompts.

### 4.4 Driving Claude Code

- **FR-17** The harness must run turns non-interactively, in the worker's repo directory, with the prompt supplied by the harness.
- **FR-18** Turns must never block waiting for a permission approval.
  - **v1 policy:** workers run with permission checks bypassed. The disposable VM and trusted inputs (§1.1) are the safety boundary.
  - The policy must be a configuration setting, so a restricted allowlist can replace it later without code changes.
  - Any permission denial that does occur must be recorded and surfaced.
- **FR-19** The first turn of an effort must establish a session. Every later turn for that effort must continue the same session.
- **FR-20** The full structured output of every turn must be saved on the host, per effort and per turn. The final outcome must be stored alongside it, including result text, exit status, cost and any permission denials.
- **FR-21** Only one turn may run per worker at a time. A second turn for a busy worker must queue or fail clearly.
- **FR-22** I must be able to SSH into any worker and continue its session interactively in Claude Code. While I'm attached, the harness must not start turns on that worker.
  - Attachment is detected through a harness-provided attach command that holds a per-worker lock. A plain `ssh` that bypasses it is not detected, and this is documented.
- **FR-23** A turn exceeding a configurable wall-clock limit or maximum turn count must be interrupted gracefully, so the session stays resumable, before any forced kill. Spend is limited by these and by the concurrency cap (NFR-1), not by dollar amounts.
- **FR-40** If a turn fails because the subscription usage limit is reached, the harness must mark affected efforts as rate-limited, queue their pending turns, and retry after the limit resets. The command center and notifications must show this state (FR-36).

### 4.5 Effort intake

- **FR-24** I must be able to start an effort from a GitHub issue reference. The harness:
  1. Spawns a worker.
  2. Builds the first prompt from the issue title and body plus a configurable instruction template: branch naming, open a draft PR, link the issue.
  3. Runs the first turn.
- **FR-25** I must be able to start an effort from a free-form prompt, for non-issue threads.
- **FR-26** The harness must keep a persistent registry of efforts: name, repo, issue, PR number, session identifier, master version, state and last activity.

### 4.6 PR feedback loop

- **FR-27** The harness must watch each active effort's PR for review comments, review summaries, conversation comments and failing CI checks. It must track which items it has already handled by ID, not by "newer than the last turn", so nothing arriving during a turn is missed or handled twice.
- **FR-28** When new feedback appears, the harness must continue that effort's session with a prompt saying there is new feedback on PR #n to read and address. A submitted review triggers a turn at once. Loose comments trigger one after a configurable quiet period, so a burst of comments produces one turn.
- **FR-29** The harness must never start a turn in response to anything authored by my GitHub account. Because the agent posts as me, this also prevents the agent from answering its own comments. Only feedback from other authors triggers turns.
  - Steering an effort myself is done through the harness (FR-35), not through PR comments.
  - Failing CI checks have no author and still trigger turns (FR-27).
- **FR-30** When a PR is merged or closed, the harness must mark the effort done and trigger teardown.

### 4.7 Visibility

- **FR-31** I must be able to list all efforts with:
  - state (running / idle / suspended / done)
  - PR link
  - last turn time and outcome
  - cumulative cost
- **FR-32** I must be able to view a readable rendering of an effort's latest turn: assistant text and tool calls.

### 4.8 Command center (intent only; detailed requirements to follow)

A native desktop application that is my main way of seeing and steering efforts. This section records intent so planners can make architecture decisions that fit it. It is not a complete specification.

- **FR-33** The command center must show all efforts and their live state, covering everything in FR-31, and update without manual refresh.
- **FR-34** It must let me drill into an effort: turn history, the readable turn rendering (FR-32), PR link and cost.
- **FR-35** It must let me perform the lifecycle operations available in the CLI: start an effort (issue or free-form), suspend, resume, tear down, and manage master versions. Sending a follow-up prompt to an existing effort is required, because it is my only way to steer an effort (FR-29). The full set of operations is TBD.
- **FR-36** It must surface events that need my attention, such as a failed turn, a permission denial, or a PR ready for review. It should do so through desktop notifications.
- **FR-37** It must provide a way to open an interactive session on a worker (FR-22) by launching an external terminal.
- **FR-38** It must be a client of the harness interface in §2.1. It holds no effort state of its own and adds no capability the CLI lacks.

Platform and technology constraints (see also NFR-7):
- It must be built with a native toolkit (such as GTK or Qt). Electron and webview-based shells are excluded. The choice of toolkit is left to planners, and may influence the harness core's language.
- In v1 it runs on the same machine as the VM host. The harness interface should not make a remote client impossible later.
- It must run well on Wayland (Hyprland on Omarchy) and on Fedora's default desktop. It must not depend on a system tray, which Hyprland does not provide by default.
- It is not a TUI.

## 5. Non-Functional Requirements

- **NFR-1 Concurrency:** support at least 3 simultaneous workers on one host, with the cap configurable and defaulting to 3. Per-worker resource defaults are set during planning, sized so 3 fit comfortably on the host.
- **NFR-2 Spawn time:** a new worker should be SSH-ready within ~30 s. Target TBD.
- **NFR-3 Disk:** a worker's disk clone should start near zero size and grow only with the effort's changes, up to a configurable maximum.
- **NFR-4 Master safety:** nothing may write to a master version that has dependent workers.
- **NFR-5 Idempotence:** every operation must be safe to retry after a failure, without corrupting state or leaking VMs.
- **NFR-6 Content-filter seam:** all externally authored text (issue bodies, PR comments, review text) must pass through a single, replaceable processing stage before it reaches a prompt. In v1 that stage does nothing. A comparable seam should exist for worker network egress. Both exist so scrubbing can be added later without re-architecting.
- **NFR-7 Host portability:** the harness and command center must install and run on both Omarchy (Arch-based) and Fedora. Packaging may differ per distribution. Dependencies must be available on both, and nothing may assume a particular distribution's paths or package manager at runtime.
- **NFR-8 Reconciliation:** the harness must survive a restart of itself or a reboot of the host without losing efforts. On start, and periodically, it must compare the registry with the VMs, disks and clones that actually exist, then adopt, resume or clean up anything that disagrees. A turn that was running when the harness died is marked interrupted and remains resumable. Orphaned VMs and disks are removed.

## 6. Accepted Risks

v1 deliberately "runs with scissors": private repos only, trusted authors only, permission checks bypassed. The risks below are accepted on that basis. If any premise in §1.1 changes (public repos, outside contributors), these must be revisited first.

- **Full credentials in every worker.** Each worker can push to any repo and act as me on GitHub. This is accepted for v1.
  - Possible mitigation: use a fine-grained GitHub token limited to the target repos.
- **Prompt injection via issue and PR text.** An agent reading text it treats as instructions runs with my credentials and without permission prompts. This is accepted because all authors are trusted.
  - Future mitigation: scrubbing through the NFR-6 seam, labeled-issue intake, and an allowlist of feedback authors.
- **No egress filtering in v1.** Workers have the same network access as my machine. The NFR-6 seam leaves room to add it.

## 7. Open Questions

None at present. Earlier questions are resolved below.

### 7.1 Decisions

| Topic | Decision |
|---|---|
| Loop prevention | Ignore everything authored by my account and react only to other authors (FR-29). |
| Permission policy | Broad autonomous action in v1, configurable (FR-18). |
| Trigger model | Manual start only in v1. |
| Idle behavior and session continuity | Suspend on idle, tear down only after a long cutoff, merge/close or by hand, and revive from the remote branch with a fresh session when feedback arrives (FR-10, FR-11, FR-39). |
| Concurrency | Default cap of 3 workers (NFR-1). On usage-limit errors, pause, queue and retry after reset (FR-40). |
| Cost guardrails | Wall-clock and max-turn limits plus the concurrency cap, with no dollar caps (FR-23). |
| Multiple repos | One master containing all target repos. |
| Master refresh | Manual in v1. |
| Command center toolkit | Native toolkit only; Electron and webview shells excluded (§4.8). |
| Command center location | Same machine as the VM host in v1, without precluding a remote client (§4.8). |
| Interactive access from the GUI | Launch an external terminal (FR-37). |

## 8. Acceptance Criteria (v1)

- [ ] Refreshing the master produces a new version and leaves existing workers unaffected.
- [ ] Killing the harness mid-spawn and mid-turn, then restarting it, leaves no orphan VMs or disks, and the interrupted effort can continue.
- [ ] A master refresh whose smoke test fails leaves the current master unchanged and cleans up the partial version.
- [ ] Three efforts can run at the same time on separate issues, each producing its own draft PR, with no interference.
- [ ] A review comment from another author on one PR causes only that effort's session to continue and push a follow-up commit.
- [ ] A comment from my own account, including the agent's replies, never starts a turn.
- [ ] I can SSH into any worker and continue its session interactively.
- [ ] Merging a PR leads to its worker being torn down. Unreferenced master versions are prunable.
- [ ] I can see the state, PR and cumulative cost of every effort at a glance.
- [ ] After a rate-limit failure, affected efforts show as rate-limited and resume on their own after the reset.
- [ ] An effort torn down for age with an open PR is revived when new feedback arrives, on the PR branch with a fresh session.
- [ ] The CLI and command center are both clients of the same harness interface. Quitting the command center changes nothing about running efforts.
- [ ] The harness and command center install and pass the above on both Omarchy and Fedora.
