# VM Agent Harness: Requirements (Draft for Review)

## 1. Purpose

Run several Claude Code efforts at the same time, each in its own clean Linux VM. Every VM is a disposable copy of a "master" VM that looks and behaves like my own dev environment: same user, dotfiles, `gh` login and Claude login. The agent works there the way I would, and I talk to it from the command center. It explores and discusses first. It branches, commits, pushes and opens PRs when I tell it to, and it tells me what it thinks of review feedback before anything is changed.

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
- A PR feedback loop: new review comments are surfaced to me, and I discuss them with the right session. An automatic mode that feeds them straight in is available as a policy.
- Lifecycle management: spawn, suspend, resume, and teardown when I decide.
- A native desktop **command center** for managing efforts (§4.8). Its detailed design is deferred, but its existence is a v1 architectural input.

### 2.1 Architectural constraints

These follow from the command center and the host platforms. Planners must account for them even where the command center itself is not yet specified.

- The harness's capabilities must be available through **one programmatic interface** that both the CLI and the command center use. No capability may exist only inside a client.
- That interface must support **live updates** (push or subscription), not only polling, so a GUI can reflect state changes as they happen. This covers the output of a running turn as well as state changes (FR-46).
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
     - `~/.claude/settings.json`
     - `~/.claude/CLAUDE.md`
     - skills, agents and shell rc as listed
     - the parts of `~/.claude.json` that workers need, handled as FR-15 describes

     Two things are not plain file copies:
     - **GitHub credential.** `gh` may keep its token in the host keyring, so `~/.config/gh` can't be relied on. The refresh must read the token from the host's `gh` login (`gh auth token`) and configure workers from it (FR-16). Nothing in a worker may depend on the host keyring.
     - **Claude Code state.** See FR-15.
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
- **FR-11** A worker is torn down, with its disk clone deleted, only when I decide it should be. Until then it persists. Idle workers are suspended (FR-10) and keep their disk and session, so I can resume one to continue the work or debug something.
  - Manual teardown, from the CLI or the command center, is always available.
  - Two automatic triggers exist as opt-in policies, both **off by default**: teardown when the PR is merged or closed, and teardown after a worker has been suspended longer than a configurable cutoff.
  - With the merge/close policy off, a merged or closed PR is recorded on the effort and raises an attention item (FR-30, FR-45). The effort and its worker are left as they are.

  Manual teardown and age-based teardown are subject to the work-preservation check in FR-41.
- **FR-39** The harness must be able to **revive** an effort whose worker was torn down for age or by hand: spawn a new worker from the current master, check out the effort's salvage branch if FR-41 created one (otherwise the PR's remote branch, otherwise the repo's default branch), and start a fresh session. Restoring the old session is not required. Before any teardown, the harness must copy the effort's session transcript to the host.
  - **Trigger.** Reviving is normally my action: I send the effort a prompt from the CLI or the command center, or use "discuss feedback" (FR-28). New feedback on a torn-down effort's open PR still raises an attention item, since the harness watches the PR, not the worker. Only under the auto feedback policy does new feedback revive the effort by itself. An effort with no PR (FR-47) has no feedback, so it is always revived by hand.
  - **Seed.** The first prompt of the new session includes the PR context if there is a PR. In every case it includes the effort's original prompt and a note saying that this is a revival, which branch was checked out, and where the previous transcript is on the host.
- **FR-41** Teardown must not silently lose work. Before removing a worker for age or by hand, the harness must check every repo in the worker for uncommitted changes, untracked files that git doesn't ignore, and commits that are not on the remote. The check may resume a suspended worker.
  - If there is nothing to save, teardown proceeds.
  - **Age-based teardown:** the harness commits any uncommitted work as a WIP commit, pushes everything to a salvage branch on the remote (for example `klank/salvage/<effort>`), and then tears down. If the push fails, the worker is kept and I am notified (FR-36). The harness retries later (NFR-5).
  - **Manual teardown:** the harness refuses by default and shows what would be lost. I can choose to save to the salvage branch and tear down, or to discard. Discarding needs explicit confirmation in the CLI and in the command center.
  - Automatic teardown on a merged or closed PR (the opt-in policy in FR-11) skips the check, since the work is merged or deliberately abandoned. It still copies the session transcript (FR-39).
  - Manual teardown of an effort whose PR is merged or closed still runs the check, since there may be work after the merge. It follows the manual rules above.
  - Build artifacts and other git-ignored files are not preserved.
- **FR-12** Per-worker vCPU, RAM and maximum disk size must be configurable, with a global default.

### 4.3 Identity and authentication

- **FR-13** Claude Code in workers must authenticate with a long-lived subscription token from `claude setup-token`, supplied via the `CLAUDE_CODE_OAUTH_TOKEN` environment variable.
- **FR-14** My interactive Claude login (`~/.claude/.credentials.json`) must **not** be copied into the master. Many VMs refreshing one shared login is avoided by design.
- **FR-15** Claude Code in workers must load my normal configuration: settings, CLAUDE.md, skills and MCP servers.
  - User-scoped MCP servers, onboarding state and per-project workspace trust live in `~/.claude.json`, not in `~/.claude/settings.json`. The master must therefore carry a `~/.claude.json` built for workers. It must not be a verbatim copy of the host's file.
  - It must include my user-scoped MCP server definitions and mark onboarding complete and each target repo path as trusted.
  - It must leave out host login and account identity (FR-14), and host-specific per-project state and history. Planners must check the exact fields against the pinned Claude Code version.
  - MCP servers that can't work in a worker, such as ones needing interactive OAuth or pointing at host-local services, must be either made to work or excluded. The refresh must report which ones were excluded and why. A silently missing server is not acceptable.
  - Note: Claude Code's "bare" mode skips this configuration and does not read `CLAUDE_CODE_OAUTH_TOKEN`, so it is not suitable here.
  - Headless operation must not be broken by first-run onboarding or workspace-trust prompts. Every master version must pass a smoke test before it becomes current: in a throwaway clone, run a trivial headless turn and confirm that settings, skills and MCP servers loaded, that no interactive prompt appeared, that `gh` is authenticated, and that git can reach every target repo's remote with those credentials (for example `git ls-remote`). A failed smoke test fails the refresh (FR-4).
- **FR-16** `gh` and git push must work in workers with my identity, with no prompts.
  - The credential comes from the token the refresh read from my host `gh` login (FR-3), not from a copy of `~/.config/gh`.
  - Where the token lives is left to planners: baked into the master, or supplied to each worker at spawn. Supplying it at spawn keeps it out of the image and lets it rotate without a refresh. Either way it falls under the credential risk in §6.

### 4.4 Driving Claude Code

- **FR-17** The harness must run turns non-interactively, in the worker's repo directory, with the prompt supplied by the harness.
- **FR-18** Turns must never block waiting for a permission approval.
  - **v1 policy:** workers run with permission checks bypassed. The disposable VM and trusted inputs (§1.1) are the safety boundary.
  - The policy must be a configuration setting, so a restricted allowlist can replace it later without code changes.
  - Any permission denial that does occur must be recorded and surfaced.
- **FR-19** The first turn of an effort must establish a session. Every later turn for that effort must continue the same session.
- **FR-20** The full structured output of every turn must be saved on the host, per effort and per turn. The final outcome must be stored alongside it, including result text, exit status, cost and any permission denials. Output must be written to the host as the turn produces it, not only when the turn ends, so a turn that is interrupted (NFR-8) or killed still leaves its output so far (FR-46).
- **FR-21** Only one turn may run per worker at a time. A second turn for a busy worker must queue or fail clearly.
- **FR-22** I must be able to SSH into any worker and continue its session interactively in Claude Code. While I'm attached, the harness must not start turns on that worker.
  - The harness must detect attachment in two ways:
    - A harness-provided attach command that holds a per-worker lock. This is the preferred route.
    - A check, made before every turn starts, for a live interactive `claude` process on the worker. This catches a plain `ssh` that bypasses the attach command.
  - If either is present, the effort is in the attached state (FR-42) and the turn is held until I detach. The held turn is queued, not dropped (FR-21).
  - An open shell with no interactive `claude` process does not count as attached. This is a known limit. If I start `claude` just after the check, the turn may already have begun, and this is documented.
- **FR-23** A turn exceeding a configurable wall-clock limit or maximum turn count must be interrupted gracefully, so the session stays resumable, before any forced kill. Spend is limited by these and by the concurrency cap (NFR-1), not by dollar amounts.
- **FR-44** I must be able to abort a running turn from the CLI and from the command center, and to drop turns that are queued for an effort.
  - An abort uses the same graceful interrupt as FR-23, so the session stays resumable. A forced kill happens only if the turn hasn't stopped within a configurable grace period.
  - After an abort the effort enters the paused state (FR-42). Queued turns are kept, and nothing starts until I resume the effort or send it a prompt. Sending a prompt counts as a harness-initiated prompt for the circuit breaker (FR-29).
  - The turn's outcome is recorded as aborted, distinct from a crash or a limit being hit. Work already done in the worker is left as it is.
- **FR-40** If a turn fails because the subscription usage limit is reached, the harness must mark affected efforts as rate-limited, queue their pending turns, and retry after the limit resets. The command center and notifications must show this state (FR-36).

### 4.5 Effort intake

- **FR-24** I must be able to start an effort from a GitHub issue reference. The harness:
  1. Spawns a worker.
  2. Builds the first prompt from the issue title and body plus a configurable instruction template.
  3. Runs the first turn.

  Efforts start **discuss-first**. The default template tells the agent to read the issue, explore the code, and tell me what it thinks and what it would propose. It must not push, open a PR or post anything to GitHub until I tell it to. The template also carries the conventions the agent follows when I do ask for a PR: branch naming, open it as a draft, link the issue.
  - There is no separate autonomous mode. If I want a PR without reviewing first, I say so to the agent, either in the start command's optional extra instruction or in a later prompt.
  - Discuss-first is an instruction to the agent, not a technical guard (§6).
- **FR-25** I must be able to start an effort from a free-form prompt, for non-issue threads. It uses the same instruction template, so it is also discuss-first.
- **FR-26** The harness must keep a persistent registry of efforts: name, repo, issue (if any), PR number (none until a PR exists, FR-47), session identifier, master version, state (FR-42) and last activity.
- **FR-42** Every effort must be in exactly one of the states below, stored in the registry. Every state change must be recorded as an event with a timestamp and a cause. The events feed live updates (§2.1), notifications (FR-36) and reconciliation (NFR-8).

  | State | Meaning |
  |---|---|
  | spawning | Worker is being created, booted and brought up to date (FR-6, FR-9). |
  | running | A turn is executing. |
  | idle | Worker is up. No turn is running or queued. |
  | attached | I am attached interactively (FR-22). No turns start. |
  | suspended | Worker is suspended with its disk intact (FR-10). |
  | rate-limited | Pending turns are held until the usage limit resets (FR-40). |
  | paused | No turns start until I resume the effort or send it a prompt. Set by the circuit breaker (FR-29), by an abort (FR-44) or by me. |
  | reviving | A new worker is being built from the PR branch (FR-39). |
  | tearing down | Worker is being removed (FR-11, FR-41). |
  | torn down | No worker. The effort can be revived (FR-39): its PR is still open, or it has no PR (FR-47). |
  | done | The PR was merged or closed, or I marked the effort done (FR-47), and the worker has been removed (FR-30). |
  | failed | Spawn or revive failed, or the worker is unrecoverable. Needs my action. |

  - The outcome of the last turn is recorded separately from the state: succeeded, failed, interrupted (NFR-8), aborted by me (FR-44), stopped at the time or turn limit (FR-23), blocked by the usage limit (FR-40), or needs input (FR-45).
  - The states spawning, running, idle, attached, suspended, reviving and tearing down have a worker. Torn down and done have none. Rate-limited and paused efforts keep their worker, which may be running or suspended. Suspending such a worker does not change the effort's state.
  - Planners must define the full set of allowed transitions. A transition that isn't defined is rejected, and reconciliation (NFR-8) resolves any mismatch between registry and reality through defined transitions only.
- **FR-47** An effort may have no PR. This is a condition of the effort, not a separate kind of effort, and applies to free-form efforts (FR-25) and to issue efforts where the agent hasn't opened a PR yet or never does.
  - **Association.** The harness must notice when a PR for the effort comes to exist, and record it in the registry as an event. Planners choose the mechanism, for example the effort marker (FR-29) in the PR body or the head branch the effort pushed. From then on the effort is treated as a PR effort in every respect.
  - **No PR means no feedback loop.** FR-27 to FR-30 apply only once a PR is associated. The harness must not poll GitHub for an effort with no PR, and no turn starts except from me.
  - **Ending.** There is no merge or close event, so only manual teardown ends the worker, or the opt-in age cutoff (FR-11) if I have turned it on, both subject to FR-41. Either leaves the effort `torn down`, which can be revived by hand (FR-39).
  - **Normal case.** Every effort starts with no PR (FR-24), and stays that way until I ask the agent for one.
  - **Mark done.** I must be able to mark a `torn down` effort, with or without a PR, as `done` from the CLI and the command center. This ends the effort for good. It can no longer be revived, and new feedback on its PR no longer revives it. I can also mark a running effort done, which tears it down first, subject to FR-41.
  - **Visibility.** The effort list (FR-31) shows "no PR" in place of the PR link. A turn that finishes on an effort with no PR raises the "waiting on me" attention item (FR-45).

### 4.6 PR feedback loop

- **FR-27** The harness must watch each active effort's PR for review comments, review summaries, conversation comments and failing CI checks. It must track which items it has already handled by ID, not by "newer than the last turn", so nothing arriving during a turn is missed or handled twice. An item counts as handled once it has been included in a prompt sent to the agent, or when I dismiss it.
- **FR-28** What happens when new feedback appears is set by a **feedback policy**, configurable globally and per repo, with a per-effort override.
  - **notify (default).** No turn starts. The harness raises an attention item (FR-45) naming the PR, the authors and the number of new items. Items that arrive close together are batched by a configurable quiet period, and the existing item is updated rather than duplicated. The command center offers a "discuss feedback" action. It sends the effort a prompt, built from a configurable template, that includes the new items and asks what the agent thinks of each and what it would do, without making changes. I can also write my own prompt instead, for example "rob left feedback on PR 123, what do you think?"
  - **auto.** The harness continues the effort's session at once with a prompt saying there is new feedback on PR #n to read and address. A submitted review triggers a turn immediately. Loose comments trigger one after the quiet period, so a burst of comments produces one turn.
  - The author allowlist (FR-43) applies under both policies.
- **FR-29** The harness must never start a turn, or raise an attention item, in response to something the agent itself posted. Because the agent posts as my GitHub account, this is decided by a marker on agent-authored content, not by author.
  - Everything the agent posts to GitHub (comments, review replies, the PR body) must carry a marker, such as a hidden HTML comment identifying the effort.
  - The harness must apply the marker itself, for example through a `gh` wrapper in the worker. It must not depend on the agent remembering to add it.
  - Feedback without the marker is handled according to the feedback policy (FR-28), including comments and reviews from my own account. This lets me steer an effort from the PR as well as through the harness (FR-35), though the harness is the main way.
  - Failing CI checks have no author and are handled by the feedback policy too (FR-27).
  - **Circuit breaker (auto policy only):** after a configurable number of consecutive feedback-triggered turns on one effort (default 5) with no harness-initiated prompt in between, the harness must pause the effort and notify me (FR-36). This bounds the damage if a marker is ever missed.
- **FR-30** When a PR is merged or closed, the harness must record that on the effort, stop watching the PR for feedback, and raise an attention item (FR-45) saying the effort is ready to tear down. If the merge/close teardown policy (FR-11) is on, it also tears the worker down. Once the worker of such an effort is removed, whether by that policy or by me, the effort is `done`.
- **FR-43** An optional **feedback author allowlist** is a prompt-injection mitigation (§6). It is unrelated to loop prevention (FR-29).
  - It is off by default. With no allowlist configured, unmarked feedback from any author is handled under the feedback policy (FR-28), since all authors are trusted in v1 (§1.1).
  - When configured, feedback from authors not on the list must neither trigger a turn nor be included in any prompt.
  - It can be set globally or per repo, and must be changeable without code changes.

### 4.7 Visibility

- **FR-31** I must be able to list all efforts with:
  - state (FR-42)
  - PR link, or "no PR" (FR-47)
  - last turn time and outcome
  - cumulative cost
  - the master version its worker is pinned to, and the disk space its worker uses (FR-48)
- **FR-45** The harness must raise an **attention item** for anything that needs me. Attention items are stored with the harness, not in a client, so they persist when the command center is closed (§2.1).
  - Each item has an effort, a type, a timestamp and a status (open or acknowledged). I can acknowledge one explicitly. Resolving the cause, such as resuming a paused effort, closes it.
  - Types:
    - turn needs input: the agent says it is blocked on me
    - turn failed, or stopped at a time or turn limit (FR-23)
    - permission denial (FR-18)
    - effort rate-limited (FR-40), paused (FR-29, FR-44) or failed (FR-42)
    - teardown blocked because the salvage push failed (FR-41)
    - PR opened or marked ready for review
    - new feedback on a PR, under the notify policy (FR-28)
    - PR merged or closed, so the effort is ready to tear down (FR-30)
    - effort is waiting on me: a turn finished and nothing else is queued
  - **Needs input:** the instruction template (FR-24) and every follow-up prompt must give the agent a convention for saying it is blocked on a decision, and the harness must detect it in the turn output. Planners choose the mechanism. A turn that ends this way is recorded with the outcome "needs input" and is not treated as an ordinary success.
  - Which types produce a desktop notification (FR-36) is configurable. By default, all of them do.
  - Open items must show in the effort list (FR-31).
- **FR-32** I must be able to view a readable rendering of any turn, finished or running: assistant text and tool calls. FR-46 presents these as a continuous conversation.
- **FR-46** Each effort must have a **conversation view**, in the command center and with CLI equivalents (for example follow and send, FR-38). This is the main way I talk to the agent.
  - It shows the whole conversation across turns, in order: my messages, and the agent's text and tool calls in the readable rendering of FR-32. Each message I send is a turn underneath (FR-17), which the view need not expose.
  - A running turn streams into the view. Text and tool calls must appear within a few seconds of being produced.
  - A client that connects, or reconnects, at any time, including mid-turn, must receive the conversation so far and then continue live, with nothing missing or duplicated. Any number of clients may view the same effort.
  - I can send a message from the view. If a turn is running, the message is queued (FR-21), shown as queued, and can be dropped (FR-44). Interrupting a running turn with a message is not required.
  - Each prompt records where it came from: me in the command center, me in the CLI, "discuss feedback" (FR-28), or the auto feedback policy.
  - **Answering a question.** An attention item of type "needs input" (FR-45) opens the conversation at the agent's question. Replying in the view closes the item.
  - Viewing does not attach (FR-22) and has no effect on the effort's state.
  - The view is built from the incrementally written host-side record (FR-20), so it survives a harness restart (NFR-8).
- **FR-48** I must be able to see where disk space goes and what each worker is pinned to, from the CLI and the command center.
  - **Per worker** (suspended ones included): the master version it is pinned to and whether that is the current master, the disk space its clone actually uses on the host, and the configured maximum (FR-12).
  - **Per master version:** whether it is current, how many workers depend on it, and its size on disk.
  - **Totals** for worker clones and for master versions.
  - Efforts without a worker (`torn down`, `done`) are listed without disk figures.
  - Pruning an unreferenced master version (FR-5) and tearing down a worker (FR-11, FR-41) can be done from this view.
  - "Actually uses" means space allocated on the host for the clone's changes, not the virtual size of the disk (NFR-3).

### 4.8 Command center (intent only; detailed requirements to follow)

A native desktop application that is my main way of seeing and steering efforts. This section records intent so planners can make architecture decisions that fit it. It is not a complete specification.

- **FR-33** The command center must show all efforts and their live state, covering everything in FR-31, and update without manual refresh.
- **FR-34** It must let me drill into an effort: turn history, the conversation view (FR-46), PR link, cost, and disk use and master pinning (FR-48).
- **FR-35** It must let me perform the lifecycle operations available in the CLI: start an effort (issue or free-form), abort a turn or drop queued turns (FR-44), suspend, resume, tear down, mark done (FR-47), use the conversation view (FR-46), see disk use and pinning (FR-48), and manage master versions. Sending a follow-up prompt to an existing effort is required. It is my main way of talking to the agent (FR-46), and GitHub is secondary. The full set of operations is whatever the CLI offers (FR-38).
- **FR-36** It must surface the attention items defined in FR-45, through desktop notifications and in the effort list. Open items raised while the command center was closed must still be shown when it starts.
- **FR-37** It must provide a way to open an interactive session on a worker (FR-22) by launching an external terminal.
- **FR-38** It must be a client of the harness interface in §2.1. It holds no effort state of its own and adds no capability the CLI lacks.

Platform and technology constraints (see also NFR-7):
- It must be built with a native toolkit (such as GTK or Qt). Electron is excluded. A lightweight webview-based shell (such as Tauri) is allowed only as a fallback, if planners show that a native toolkit is a poor fit for a specific requirement such as the turn rendering in FR-32. The choice of toolkit is left to planners, and may influence the harness core's language.
- In v1 it runs on the same machine as the VM host. The harness interface should not make a remote client impossible later.
- It must run well on Wayland (Hyprland on Omarchy) and on Fedora's default desktop. It must not depend on a system tray, which Hyprland does not provide by default.
- It is not a TUI.

## 5. Non-Functional Requirements

- **NFR-1 Concurrency:** support at least 3 simultaneous workers on one host, with the cap configurable and defaulting to 3. Per-worker resource defaults are set during planning, sized so 3 fit comfortably on the host.
- **NFR-2 Spawn time:** a new worker should be SSH-ready within about 30 s of the spawn command. This is a target, not a pass/fail gate, until it has been measured (§7). The first-boot fetch (FR-9) is not included.
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
  - v1 mitigation, off by default: the optional feedback author allowlist (FR-43).
  - Future mitigation: scrubbing through the NFR-6 seam and labeled-issue intake.
- **Discuss-first is an instruction, not a guard.** Nothing technical stops the agent from pushing or posting to GitHub before I say so. The instruction template (FR-24) is the only control in v1.
  - Possible later guard: have the `gh` wrapper (FR-29) hold outbound posts until I release them.
- **No egress filtering in v1.** Workers have the same network access as my machine. The NFR-6 seam leaves room to add it.

## 7. Open Questions

Questions that need an answer from me:

1. **Command center detail.** §4.8 records intent only. Detailed requirements (screens, interactions, the turn rendering) still have to be written. They can follow the harness requirements, but §2.1 must hold in the meantime.
2. **Spawn time target (NFR-2).** Is 30 s the right target, or should it be a hard limit? It should be settled once real spawn times are measured.

### 7.1 Deferred to planners

These are decided during planning. The requirements set the constraints and leave the choice open.

- Command center toolkit, and whether it affects the core's language (§4.8).
- Per-worker resource defaults, sized so 3 workers fit comfortably on the host (NFR-1, FR-12).
- Exact `~/.claude.json` fields to carry into the master (FR-15).
- Where the GitHub token lives: in the master, or supplied to each worker at spawn (FR-16).
- The full set of allowed state transitions (FR-42).
- The mechanism for the "needs input" convention (FR-45).
- Whether the content-filter seam (NFR-6) also enforces the author allowlist (FR-43).
- Whether the attach check counts anything beyond an interactive `claude` process (FR-22).

### 7.2 Decisions

| Topic | Decision |
|---|---|
| Loop prevention | Mark everything the agent posts and ignore marked content. Unmarked feedback, including from my own account, is handled under the feedback policy. Under auto, a circuit breaker backstops a missed marker (FR-29). |
| Feedback author filtering | Optional allowlist, off by default, as a prompt-injection mitigation only (FR-43). |
| Permission policy | Broad autonomous action in v1, configurable (FR-18). |
| Trigger model | Manual start only in v1. |
| Efforts with no PR | No PR is a condition, not a kind of effort. Every effort starts without one. No feedback loop until a PR exists. Ends by manual teardown, revived by hand, and can be marked done (FR-47). |
| Where I talk to the agent | The command center's conversation view is the main channel. GitHub is secondary (FR-46). |
| Starting an effort | Discuss-first. The agent opens a PR only when I tell it to. No separate autonomous mode (FR-24). |
| PR feedback | Notify by default. I ask the agent what it thinks before anything is changed. Auto is an opt-in policy (FR-28). |
| Teardown | Manual by default. Merge/close and age teardown are opt-in policies, off by default (FR-11). |
| Idle behavior and session continuity | Suspend on idle and keep workers until I tear them down. A torn-down effort can be revived from its branch with a fresh session (FR-10, FR-11, FR-39). |
| Concurrency | Default cap of 3 workers (NFR-1). On usage-limit errors, pause, queue and retry after reset (FR-40). |
| Cost guardrails | Wall-clock and max-turn limits plus the concurrency cap, with no dollar caps (FR-23). |
| Multiple repos | One master containing all target repos. |
| Master refresh | Manual in v1. |
| Command center toolkit | Native toolkit preferred; Electron excluded; a lightweight webview shell is an allowed fallback only if native proves a poor fit (§4.8). |
| Command center location | Same machine as the VM host in v1, without precluding a remote client (§4.8). |
| Interactive access from the GUI | Launch an external terminal (FR-37). |

## 8. Acceptance Criteria (v1)

Each criterion says what to do and what observable result passes it. "Observable" means something visible in the registry, the event log, the host-side turn records, CLI or command center output, the host's VM and disk listings, or GitHub. Nothing is judged by feel.

**Fixtures.** A private test repo with a small test suite (including a test that binds a fixed port) and a build step. A second GitHub account, used as an "other author". Test hooks that let a test shorten the idle, quiet-period and age timers, stub a usage-limit failure, and disable the marker wrapper (FR-29). Anything that depends on live model behaviour is also tested against canned turn output, so the harness logic is verified without relying on the model.

### 8.1 Master and workers

- [ ] **AC-1 Refresh leaves workers alone** (FR-3, FR-5, NFR-4)
  - Do: with one worker running from master v1 and mid-turn, refresh the master.
  - Pass when: master v2 exists and is current. v1 still exists and is unchanged. The worker finishes its turn, and the registry still shows it on v1.
- [ ] **AC-2 Failed smoke test** (FR-4, FR-15)
  - Do: make the smoke test fail (for example, an invalid GitHub token), then refresh.
  - Pass when: the refresh reports failure with the reason. The current master has the same version and image checksum as before. No partial version appears in the version list or on disk.
- [ ] **AC-3 Worker identity** (FR-6, FR-7, FR-8)
  - Do: spawn two workers from the same master version.
  - Pass when: both are reachable as `ssh <worker-name>`. Hostnames, MAC addresses and SSH host keys all differ.
- [ ] **AC-4 Fresh code** (FR-9)
  - Do: push a new commit to the test repo's default branch after the master was built, then spawn a worker.
  - Pass when: before the first turn starts, the worker's repo is at that commit.
- [ ] **AC-5 Credentials** (FR-13, FR-14, FR-16)
  - Do: in a new worker, check `gh auth status`, run a `git push` to the test repo, and look for `~/.claude/.credentials.json`.
  - Pass when: `gh` is authenticated, the push succeeds without any prompt, and the credentials file does not exist. A headless turn runs using `CLAUDE_CODE_OAUTH_TOKEN`.
- [ ] **AC-6 Idle suspend and resume** (FR-10)
  - Do: set a short idle period and leave an effort idle. Then send it a prompt.
  - Pass when: the effort reaches `suspended` and its disk is kept. The prompt resumes the worker automatically and continues the same session ID.
- [ ] **AC-7 Merge, teardown and pruning** (FR-5, FR-11, FR-30)
  - Do: (a) with the merge/close teardown policy off, merge an effort's PR. (b) Tear that effort down by hand. (c) Turn the policy on and merge a second effort's PR. (d) Try to prune the master version they used, first while another worker still uses that version and again once none does.
  - Pass when: in (a), within one poll interval the effort records the merged PR and an attention item is open, while the worker, its disk and its state are unchanged, and later comments on the merged PR raise nothing. In (b), the worker and disk are gone, the transcript is on the host and the state is `done`. In (c), within one poll interval the worker and disk are gone, the state is `done` and the transcript is on the host. In (d), pruning is refused while a worker depends on the version and succeeds when none does.

### 8.2 Turns and sessions

- [ ] **AC-8 Turn records** (FR-17, FR-19, FR-20)
  - Do: run a first turn and a follow-up turn on an effort.
  - Pass when: the host has the full structured output for each turn, per effort and per turn. Each has an outcome record with result text, exit status, cost and permission denials. Both turns share one session ID.
- [ ] **AC-9 Turn limits** (FR-23)
  - Do: set a very short wall-clock limit and give the effort a long task. Then send a follow-up prompt.
  - Pass when: the turn is interrupted before any forced kill, its outcome is "stopped at limit", and the follow-up continues the same session.
- [ ] **AC-10 Abort** (FR-44)
  - Do: start a turn, queue a second one, then abort the first.
  - Pass when: the turn stops gracefully with outcome "aborted". The effort is `paused` and the second turn is still queued. Sending a prompt starts a turn in the same session.
- [ ] **AC-11 Attach** (FR-22)
  - Do: (a) attach with the attach command. (b) Separately, `ssh` in directly and start an interactive `claude`. In each case, send the effort a prompt from the CLI while attached. Then detach.
  - Pass when: in both cases the effort shows `attached`, the turn is held and queued, and it starts after detach. The session transcript shows no interleaved writes.
- [ ] **AC-12 Rate limit** (FR-40)
  - Do: stub a usage-limit failure on a turn, with a reset time a short way ahead.
  - Pass when: the effort shows `rate-limited` and I am notified. Pending turns are queued. After the stubbed reset, the turns run with no action from me.
- [ ] **AC-13 Needs input** (FR-45, FR-36)
  - Do: (a) feed the harness canned turn output that uses the needs-input convention. (b) Run one live turn on a task that cannot proceed without a decision from me. In both cases, keep the command center closed until after the turn ends.
  - Pass when: the outcome is "needs input", an attention item is open, and a notification was sent. When the command center starts, it shows the open item. Replying to the question in the conversation view starts a turn in the same session and closes the item.

### 8.3 Concurrency and the feedback loop

AC-15 to AC-19 run with the feedback policy set to **auto**. AC-35 covers the default **notify** policy.

- [ ] **AC-14 Three concurrent efforts** (FR-24, NFR-1)
  - Do: start three efforts from three issues at the same time. When each has finished its first turn, tell each to open a draft PR.
  - Pass when: the first-turn records show overlapping time windows for all three, and GitHub shows no PR, push or comment from any of them yet. After the instruction, each effort has its own worker and its own draft PR linked to its issue, and the follow-up turns also overlap. Each PR contains only its own effort's commits. The fixed-port test passes in all three.
- [ ] **AC-15 Feedback reaches the right session** (FR-27, FR-28)
  - Do: with three efforts in flight, have the other author submit a review on PR A.
  - Pass when: only effort A runs a new turn, in its existing session. A follow-up commit appears on PR A's branch. B and C have no new turns.
- [ ] **AC-16 Each item handled once** (FR-27, FR-28)
  - Do: post three loose comments in quick succession, then one more during the resulting turn.
  - Pass when: the first three produce one turn after the quiet period. The fourth is handled by a later turn. The turn records show no comment ID handled twice and none missed.
- [ ] **AC-17 Loop prevention** (FR-29)
  - Do: have the agent reply to a review comment. Then post an unmarked comment from my own account.
  - Pass when: the agent's reply carries the marker, and no turn starts from it within twice the quiet period. My unmarked comment starts a turn.
- [ ] **AC-18 Circuit breaker** (FR-29)
  - Do: disable the marker wrapper, set the breaker threshold to 2, and have the agent comment on its own PR each turn.
  - Pass when: after two consecutive feedback-triggered turns the effort is `paused` and an attention item is raised.
- [ ] **AC-19 Author allowlist** (FR-43)
  - Do: with no allowlist, post a comment as the second account. Then configure an allowlist without that account and post another.
  - Pass when: the first comment starts a turn. The second neither starts a turn nor appears in any later prompt.

### 8.4 Lifecycle safety and recovery

- [ ] **AC-20 Crash recovery** (NFR-5, NFR-8)
  - Do: in separate runs, kill the harness with SIGKILL (a) during spawn and (b) during a turn. Restart it each time.
  - Pass when: after reconciliation, every VM and disk on the host is referenced by the registry. For (a), the effort either finishes spawning or is cleaned up through a defined transition. For (b), the last turn shows "interrupted", and a follow-up prompt continues the same session.
- [ ] **AC-21 Age teardown saves work** (FR-11, FR-41)
  - Do: turn the age teardown policy on. Give an effort an open PR, one unpushed commit and one uncommitted change. Let it exceed the age cutoff.
  - Pass when: a salvage branch on the remote contains both pieces of work. The worker and disk are gone, the transcript is on the host, and the state is `torn down`.
- [ ] **AC-22 Revive** (FR-39)
  - Do: after AC-21, post new feedback from the other author, first with the feedback policy on notify and then on auto.
  - Pass when: under notify, an attention item is raised and no worker is created until I use "discuss feedback", which then takes the effort through `reviving` to a new worker. Under auto, the effort goes through `reviving` by itself. In both cases the new worker is on the salvage branch, the session ID is new, and the first prompt includes the PR context.
- [ ] **AC-23 Salvage failure** (FR-41)
  - Do: make the remote unreachable, then let an age teardown run on a dirty worker.
  - Pass when: the worker is kept, an attention item is raised, and teardown is retried once the remote is reachable.
- [ ] **AC-24 Manual teardown** (FR-41)
  - Do: tear down a dirty worker by hand.
  - Pass when: the tool refuses and lists what would be lost. Save-and-tear-down puts the work on the salvage branch. Discard works only after explicit confirmation.
- [ ] **AC-25 One state, always** (FR-42)
  - Do: run the scenarios above and read the event log.
  - Pass when: at every point each effort is in exactly one defined state. Every state change appears as an event with a timestamp and cause. After a restart, the registry matches the VMs that exist.

- [ ] **AC-33 Effort with no PR** (FR-47, FR-39, FR-41)
  - Do: turn the age teardown policy on. Start a free-form effort whose turn commits locally and opens no PR. Check the list. Let it exceed the age cutoff. Then send it a prompt. Separately, start a second free-form effort and tear it down by hand while it is clean.
  - Pass when: the list shows "no PR" and a "waiting on me" item after the turn. The harness makes no GitHub feedback queries for the effort. After the age cutoff, a salvage branch holds the commit and the state is `torn down`. The prompt moves the effort through `reviving` to a new worker on the salvage branch, with a new session ID and the original prompt in the first prompt. The clean effort revives onto the default branch, since no salvage branch exists. Marking an effort done afterwards leaves it `done`, and sending it a prompt is then refused.
- [ ] **AC-34 PR appears later** (FR-47, FR-27)
  - Do: on an effort with no PR, have a follow-up turn open a PR. Then have the other author comment on it.
  - Pass when: an association event is recorded and the list shows the PR link. The comment starts a turn in the same session. No such comment-driven turn happened before the PR existed.

### 8.5 Visibility and the command center

- [ ] **AC-26 List view** (FR-31, FR-45, FR-47, FR-48)
  - Do: with efforts in several states, including one with no PR and one suspended, run the CLI list command and open the command center.
  - Pass when: a single view shows, for every effort, the state, PR link (or "no PR"), last turn time and outcome, cumulative cost, pinned master version, disk use, and open attention items. No per-effort command is needed.
- [ ] **AC-27 Live updates** (FR-33)
  - Do: with the command center open, change an effort's state from the CLI.
  - Pass when: the command center shows the change within 5 seconds with no manual refresh.
- [ ] **AC-28 Client independence** (§2.1, FR-38)
  - Do: quit the command center during a running turn. Then do every FR-35 operation from the CLI.
  - Pass when: the turn completes and the registry and events are the same as in a run without the command center. Every operation the command center offers is also available in the CLI.
- [ ] **AC-29 Desktop fit** (§4.8, NFR-7)
  - Do: start the command center on Hyprland on Omarchy and on Fedora's default desktop, with no system tray.
  - Pass when: it runs, and attention notifications arrive on both.

- [ ] **AC-32 Conversation view** (FR-46, FR-20)
  - Do: replay a canned slow turn that emits text and tool calls over about 60 seconds. (a) Open the effort's conversation view from the start. (b) Open it in a second client 30 seconds in. (c) Send a message while the turn is running. (d) Kill the harness with SIGKILL at 40 seconds, restart it and open the view again.
  - Pass when: in (a), items appear within a few seconds of being emitted. In (b), the conversation so far appears, then continues live, with nothing missing or duplicated against the host-side record. In (c), the message shows as queued, then runs after the turn ends in the same session, and the turn record shows it came from the command center. In (d), the output up to the kill is still on the host, and the turn is marked "interrupted" (NFR-8). Viewing changes neither the effort's state nor the turn records.
- [ ] **AC-35 Feedback under notify** (FR-27, FR-28, FR-29)
  - Do: with the default notify policy, have the other author submit a review on PR A and post three loose comments. Then use "discuss feedback" in the command center. Then post one more comment, and have the agent reply with a marked comment.
  - Pass when: no turn starts from the feedback. There is a single attention item for A covering the review and comments, updated rather than duplicated. B and C have none. "Discuss feedback" sends a prompt in A's existing session that includes exactly those items and asks for an assessment without changes. No commit appears. The later comment raises a new item that does not repeat the earlier ones. The agent's marked reply raises nothing.
- [ ] **AC-36 Storage and pinning** (FR-48, FR-5)
  - Do: with workers pinned to two master versions, one of them suspended and one effort torn down, open the storage view in the CLI and the command center. Have one worker write a large file.
  - Pass when: each worker shows its pinned version, whether that is current, and the disk space its clone uses, which matches the host's allocated size for the clone and grows after the write. Each master version shows whether it is current, its dependents and its size. The torn-down effort is listed without disk figures. Pruning a referenced version is refused, and an unreferenced one can be pruned from the view.

### 8.6 Hosts and measurements

- [ ] **AC-30 Both distros** (NFR-7)
  - Do: install on a clean Omarchy host and a clean Fedora host using the documented steps, then run all the criteria above.
  - Pass when: every criterion passes on both.
- [ ] **AC-31 Measurements** (NFR-2, NFR-3)
  - Do: spawn 10 workers one after another. Record the time to SSH-ready and each fresh clone's allocated size against its master.
  - Pass when: the median and worst times and the sizes are recorded. This is informational. It sets the target in §7.
