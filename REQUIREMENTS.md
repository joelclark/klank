# VM Agent Harness: Requirements (Draft for Review)

## 1. Purpose

Run several Claude Code efforts at the same time, each in its own clean Linux VM. Every VM is a disposable copy of a "master" VM for its repo that looks and behaves like my own dev environment: same user, dotfiles, `gh` login and Claude login. The agent works there the way I would, and I talk to it from the command center. It explores and discusses first. It branches, commits, pushes and opens PRs when I tell it to, and it tells me what it thinks of review feedback before anything is changed. When I want a second opinion, such as a review, I can bring another agent such as Codex into an effort as a consultant. A consultant reads and advises. It never changes the code or posts anything.

The goal is **parallelism with isolation**. Separate working trees, builds and test runs mean efforts never collide.

How these requirements are met (languages, tools, structure) is left to the implementer, within the constraints in §2.1 and §5.

### 1.1 Premises

- **A personal tool.** It has one user, me, on my own host. Nothing has to support other users, shared hosts, or installation by anyone else. Setup must still be documented and repeatable (FR-2), because I install it on more than one host and distribution.
- **Full VMs are a deliberate choice**, not a conclusion derived from a comparison with worktrees or containers. Planners should not re-litigate it.
- **Trusted inputs for v1.** Efforts run against private repos, and every author of issues, PRs and comments is trusted. This is what makes the v1 security posture in §6 acceptable. The design must leave room to tighten it later (NFR-6).
- **Tokens are for judgement.** The harness does everything deterministic itself: booting, fetching, installing dependencies, migrating and seeding, starting the app, watching GitHub, detecting attachment and retrying after a rate limit. It does not leave these to the agent, because each one would cost tokens and usage, and some would be done inconsistently. The agent is told what has already been done (FR-16) so it doesn't repeat it.
- **A native desktop command center** (§4.10) is part of the product. It is detailed later, but it affects architecture now (§2.1).

## 2. Scope

### In scope (v1)
- A single Linux host with KVM. Supported host distributions: **Omarchy** (Arch-based, Wayland/Hyprland) and **Fedora**.
- Setup on a clean host, and configuration through text files (§4.1).
- Target repos, each described by a repo definition committed in the repo, which my configuration can supply or replace (§4.1).
- A master image per target repo, versioned and refreshable, built from the repo's definition. The guest distribution is set per repo. v1 supports at least one, with room for more.
- Worker VMs created as copy-on-write clones of a specific master version.
- Driving Claude Code inside workers non-interactively from the host, plus manual interactive access.
- A PR feedback loop: new review comments are surfaced to me, and I discuss them with the right session. An automatic mode that feeds them straight in is available as a policy.
- Consultant agents: other coding agents I bring into an effort by hand, for a review or a second opinion (§4.8).
- Lifecycle management: spawn, suspend, resume, and teardown when I decide.
- A native desktop **command center** for managing efforts (§4.10). Its detailed design is deferred, but its existence is a v1 architectural input.

### 2.1 Architectural constraints

These follow from the command center and the host platforms. Planners must account for them even where the command center itself is not yet specified.

- The harness's capabilities must be available through **one programmatic interface** that both the CLI and the command center use. No capability may exist only inside a client.
- That interface must support **live updates** (push or subscription), not only polling, so a GUI can reflect state changes as they happen. This covers the output of a running turn as well as state changes (FR-55).
- The harness must keep running independently of any client. Closing the command center must not stop, or change the state of, any effort.
- Machine-readable state (registry, turn records, events) is the source of truth. Anything a client displays must be derivable from it.

### Out of scope (v1)
- Other users: shared hosts, several users on one host, or installation for anyone but me (§1.1).
- Efforts that span more than one repo. Every effort belongs to one target repo.
- Cloning a *running* master (memory snapshots). Workers boot fresh from disk.
- Multiple hosts or a cluster scheduler.
- macOS or Windows guests.
- Credential isolation between workers and my account. This is an accepted risk; see §6.
- A web UI, and any Electron-based client.
- Input scrubbing and egress filtering. Deferred, but a seam is required (NFR-6).
- Any agent other than Claude Code as an effort's driver.
- Consultants that change code, commit, push or post to GitHub, and consults started by the driver or by a policy rather than by me (§4.8).
- Forwarding worker ports to `localhost` on the host, for apps that need a `localhost` origin (for example real OAuth callbacks). Apps are reached by worker name (FR-13). Repos that need OAuth use a local dev bypass.

## 3. Terms

| Term | Meaning |
|---|---|
| **Target repo** | A GitHub repo listed in my harness configuration, so efforts can be started in it (FR-4). |
| **Repo definition** | What a target repo needs: guest distribution, toolchain, services, prep steps and app. Committed in the repo, or supplied by my harness configuration (FR-4). |
| **Harness configuration** | My own settings: target repos, dotfile lists, pinned versions, consultants, policies, templates and limits (FR-3). |
| **Master** | A versioned base disk image for one target repo: its guest distribution and toolchain, my user, tools, dotfiles and a clone of the repo. Never modified once created. |
| **Current master** | For each target repo, the master version that new workers for that repo are built from. |
| **Worker** | A VM booted from a copy-on-write clone of one specific master version. |
| **Effort** | One unit of work in one target repo, usually one GitHub issue or one thread. One effort maps to one worker. |
| **Driver** | The Claude Code agent that does an effort's work and owns its branch and PR. Every effort has exactly one. |
| **Consultant** | Another coding agent, such as Codex, that I bring into an effort to review or advise. It is read-only (§4.8). |
| **Agent** | The driver or a consultant. |
| **Turn** | One non-interactive agent run inside a worker. A driver turn, or a consult. Unqualified, "turn" covers both. |
| **Consult** | One consultant turn, started by me (FR-47). |
| **Session** | An agent's conversation within an effort, continued across its turns. The driver has one session per effort (FR-28). Each consultant has its own session per effort (FR-48). Unqualified, "session" means the driver's. |
| **Command center** | The native desktop application I use to monitor and manage efforts. See §4.10. |

## 4. Functional Requirements

### 4.1 Setup and configuration

This section covers getting the harness running on a host, adding repos to it, and configuring it.

- **FR-1** The harness must be able to check that the host is ready. Each problem it finds is reported together with what to do about it. I can run the check at any time, and it runs as part of first-time setup (FR-2). It covers at least:
  - KVM is available and usable by my user
  - the harness's dependencies are installed
  - free disk space, with a warning when it is low
  - worker names resolve on the host (FR-13)
  - `gh` is logged in on the host (FR-9)
  - the secrets the configuration refers to are present (FR-5)
  - the configuration is valid (FR-3)
- **FR-2** On a clean supported host (NFR-7), getting from installation to a first effort must follow a documented sequence of steps:
  1. Install the harness.
  2. Initialise: create a starter harness configuration and run the host check (FR-1).
  3. Provide the secrets (FR-5): the token from `claude setup-token` (FR-22) and any consultant credentials (FR-46).
  4. Add a target repo (FR-4).
  5. Build that repo's first master (FR-9).
  6. Start an effort (FR-35, FR-36).

  Each step must report what it did and what comes next. Re-running a step must be safe (NFR-5).
- **FR-3** Configuration comes from two places:
  - **Harness configuration** describes me and how I want the harness to behave: the target repos, the dotfile and config lists (FR-9), the pinned Claude Code version (FR-8), consultants (FR-45), policies, templates, timers, limits, the author allowlist and the concurrency cap. Per-repo settings of these, such as a repo's feedback policy (FR-41), belong here too, not in the repo.
  - **Repo definitions** describe what each repo needs (FR-4).

  Both must be human-editable text files, so I can keep my harness configuration in git with my dotfiles.
  - **Validation.** The harness must validate configuration before acting on it. Invalid configuration is rejected with errors that name the file and the setting. A change that fails validation is not applied, and the harness carries on with the last valid configuration.
  - **Build and runtime settings.** Every setting is one of two kinds, and the documentation says which kind each one is.
    - *Build settings* are baked into a master. They take effect only in masters built after the change. They include the guest distribution, the toolchain, the dotfile and config lists, pinned versions, consultants, and the master-time prep steps.
    - *Runtime settings* take effect without a new master, for whatever happens after the change. They include policies, templates, timers, limits, the allowlist, the concurrency cap, resources for new workers (FR-21), and the parts of a repo definition used to prepare a worker (FR-16).

    Existing workers keep the build settings of their master version (FR-12). For each target repo, the harness must show whether its current master is out of date with the build settings (FR-9).
- **FR-4** A target repo is a GitHub repo listed in my harness configuration. Every effort belongs to exactly one target repo, and each target repo has its own masters (FR-8).
  - **Repo definition.** Each target repo has a definition, committed in the repo at a fixed path. It states:
    - the guest distribution (FR-6)
    - the toolchain: languages, their versions, and system packages. Version files the repo already uses, such as `.tool-versions`, may be honoured instead of repeated.
    - the services, prep steps and app, with the app's ports and health check (FR-16)
    - optional master-time prep steps (FR-9)
    - anything else the repo needs to build and test (FR-8). Secrets are referred to by name and never committed (FR-5).
  - **Local override.** My harness configuration may supply a repo's definition. Use it to try a repo before committing a definition, or for a repo where I don't want to add one. When the repo also has a committed definition, the override replaces it as a whole. The harness shows when an override is in effect.
  - **Which revision.** A master build reads the definition from the repo's default branch. When an override is in effect, it is used instead, for master builds and worker preparation alike.
    - Preparing a worker (FR-16) reads the definition from the worker's own checkout, after the fetch (FR-15). An effort whose branch changes its prep steps or app is therefore prepared with its own version.
    - A worker can't pick up build settings changed on its effort's branch, because its master was built from the default branch. When the harness prepares a worker and finds that the build settings in its checkout differ from those its master was built with, it records an event and raises an attention item (FR-53).
  - **Adding a repo.** I add a repo by listing it in my harness configuration and building its first master. Until the repo has a master, efforts for it are refused, with the reason (FR-35, FR-36).
  - **Removing a repo.** Taking a repo out of the configuration stops new efforts for it. Existing efforts and their workers are unaffected. Its master versions become prunable once no worker depends on them (FR-11).
- **FR-5** Secrets the harness needs on the host must be kept in one place on the host, readable only by my user. These are the Claude token (FR-22), consultant credentials (FR-46), and private registry credentials and env files (FR-8).
  - The harness configuration and repo definitions refer to secrets by name and never contain them.
  - The GitHub token is the exception. It is read from my host `gh` login at refresh (FR-9).
  - Rotating a secret must not require editing configuration. Whether a rotated secret reaches existing workers without a new master depends on where the secret lives (FR-25, FR-46).
  - A missing secret is reported by the host check (FR-1) and by the step that needs it.
- **FR-6** The guest distribution is set by each repo's definition, from a list of distributions the harness supports.
  - v1 must support at least one (§7).
  - Adding another distribution must not need changes outside the part of the harness that is specific to that distribution, as with consultants (FR-45).
  - Validation rejects a repo definition that names an unsupported distribution (FR-3).
  - My dotfiles and config (FR-9) must work on every guest distribution I use. Making them work is my job, not the harness's.

### 4.2 Master image management

- **FR-7** The master must have a user with the same username as my host account, so my home directory has the same path on the host and in every worker.
  - Absolute paths in the configuration I bring over (settings, hooks, MCP config, `CLAUDE.md`) and in per-project data must resolve identically in a worker.
  - Every file brought in from the host must end up owned by this user, with correct permissions. A matching UID is not required.
- **FR-8** Each target repo has its own masters (FR-4). A master must contain:
  - The repo's guest distribution and toolchain, from its definition (FR-4, FR-6).
  - Claude Code, pinned to a specific version with auto-update disabled, `gh` and `git`.
  - The CLI of each configured consultant, pinned and with auto-update disabled in the same way (FR-46).
  - A clone of the repo.
  - Everything the repo needs to build and test, from its definition: private package registry credentials, env files, and local services such as databases or Docker. Anything secret here falls under the credential risk in §6.
- **FR-9** Refreshing a repo's master must produce a **new** master version. It must never modify an existing version in place.
  - **Two kinds.** A refresh is either a full build or an incremental refresh:
    - A **full build** starts from a base image of the repo's guest distribution. It installs the toolchain, Claude Code, `gh`, `git` and the consultants, creates my user (FR-7), clones the repo, and then runs the steps below.
    - An **incremental refresh** starts from the repo's current master version and runs only the steps below.
  - **Which kind.** A repo's first master is always a full build. After that, a full build is required whenever a build setting (FR-3) has changed since the current version was built. Otherwise an incremental refresh is enough.
    - I refresh with one operation. The harness chooses the kind, and says which it chose and why. I can force a full build.
    - I can refresh one repo's master or all of them. A change to a setting shared by every master, such as the pinned Claude Code version, needs all of them refreshed.
    - Refreshing stays manual in v1 (§7.2).

  Both kinds must:
  1. Bring in my current dotfiles and config from the host, from a configurable list. The defaults are:
     - `~/.gitconfig`
     - `~/.claude/settings.json`
     - `~/.claude/CLAUDE.md`
     - skills, agents and shell rc as listed
     - the parts of `~/.claude.json` that workers need, handled as FR-24 describes
     - each consultant's configuration, from its own list (FR-46)

     Three things are not plain file copies:
     - **GitHub credential.** `gh` may keep its token in the host keyring, so `~/.config/gh` can't be relied on. The refresh must obtain the token from the host's `gh` login (for example with `gh auth token`) and configure workers from it (FR-25). Nothing in a worker may depend on the host keyring.
     - **Claude Code state.** See FR-24.
     - **Consultant credentials.** See FR-46.
  2. Update the repo to the latest default branch, as my user.
  3. Run the optional master-time prep steps from the repo definition (FR-4), such as installing dependencies.
  4. Make the new version current for that repo atomically.
- **FR-10** A failed refresh must leave the repo's current master untouched and clean up any partial version.
- **FR-11** A master version must not be deleted while any worker depends on it. Unreferenced versions must be prunable.

### 4.3 Worker lifecycle

- **FR-12** Spawning a worker must clone a **concrete** master version of the effort's repo, never a moving "current" pointer, then boot it and wait until it is reachable over SSH.
- **FR-13** Workers must be reachable from the host as `ssh <worker-name>`, and by the same name for any other protocol. A server listening in a worker must be reachable from the host at `<worker-name>:<port>`, for example `http://<worker-name>:3000` in a host browser.
  - The name must resolve for every program on the host, not only through an `ssh` alias, on both supported distributions (NFR-7).
  - Every worker has its own set of ports. Two workers running the same app on the same port must not collide.
  - Known limit: a plain `http://<worker-name>` origin is not a browser "secure context", which `localhost` is. Apps that need one are not supported in v1.
- **FR-14** Each worker must get its own hostname, MAC address, SSH host keys and machine-id.
- **FR-15** On first boot, a worker must fetch the latest changes in its repo before the first turn. A stale master must only cost time, never produce wrong code.
- **FR-16** The harness, not the agent, must prepare each worker's environment and start its app. This is defined in the repo definition (FR-4), read from the worker's checkout, and is what lets me log into the app in a worker and test a feature without spending a turn on setup.
  - **Definition.** In order:
    1. Services the app needs, such as a database or Docker.
    2. Prep steps, such as installing dependencies, migrating and seeding. This is the per-worker counterpart of the master-time prep in FR-9. Prep the master already did must not be repeated when its inputs, such as the lockfile, haven't changed since the master was built.
    3. The app: its start command, the ports it serves, and a health check.
  - **When.** After the fetch in FR-15 and before the first turn. The same runs for a revived effort (FR-19) on its new worker. After any resume or boot, the harness makes sure the app is up before it runs a turn.
  - **The app outlives turns.** It is managed independently of any turn or SSH session. It keeps running between turns, restarts if it crashes, and is brought back after a suspend and resume (FR-17).
  - **Failures.** A failed prep step is recorded as an event, with its log. The effort goes to `failed` (FR-38) and an attention item is raised (FR-53), unless the repo's definition says to continue past failure. FR-38 says how an effort leaves `failed`. An app that does not become healthy raises an attention item but does not stop the effort.
  - **Logs and control.** Prep and app output is saved on the host and can be viewed from the CLI and the command center. I can restart the app, or re-run prep and restart the app, from both. Neither costs a turn.
  - **The agent is told.** The first prompt of every new session, including a revival, says what was done: the environment is prepared, the app is running at its URL, where its logs are, and how to restart it. This keeps the agent from starting its own copy.
  - Auth for the app is the repo's concern. A local dev bypass in the repo's own config, seed or environment is expected to be enough.
- **FR-17** Workers that have been idle for a configurable period must be suspended, keeping their disk (and so their session) intact, and resumable on demand. Sending a turn to a suspended worker must resume it automatically.
- **FR-18** A worker is torn down, with its disk clone deleted, only when I decide it should be. Until then it persists. Idle workers are suspended (FR-17) and keep their disk and session, so I can resume one to continue the work or debug something.
  - Manual teardown, from the CLI or the command center, is always available.
  - Two automatic triggers exist as opt-in policies, both **off by default**: teardown when the PR is merged or closed, and teardown after a worker has been suspended longer than a configurable cutoff.
  - With the merge/close policy off, a merged or closed PR is recorded on the effort and raises an attention item (FR-43, FR-53). The effort and its worker are left as they are.

  Manual teardown and age-based teardown are subject to the work-preservation check in FR-20.
- **FR-19** The harness must be able to **revive** an effort whose worker was torn down for age or by hand: spawn a new worker from the repo's current master, check out the effort's salvage branch if FR-20 created one (otherwise the PR's remote branch, otherwise the repo's default branch), and start a fresh session. Restoring the old session is not required. Before any teardown, the harness must copy the effort's session transcripts to the host: the driver's and every consultant's. Consultant sessions also end at teardown. The first consult after a revival starts a new consultant session (FR-48).
  - **Trigger.** Reviving is normally my action: I send the effort a prompt from the CLI or the command center, or use "discuss feedback" (FR-41). New feedback on a torn-down effort's open PR still raises an attention item, since the harness watches the PR, not the worker. Only under the auto feedback policy does new feedback revive the effort by itself. An effort with no PR (FR-39) has no feedback, so it is always revived by hand.
  - **Seed.** The first prompt of the new session includes the PR context if there is a PR. In every case it includes the effort's original prompt and a note saying that this is a revival, which branch was checked out, and where the previous transcript is on the host.
- **FR-20** Teardown must not silently lose work. Before removing a worker for age or by hand, the harness must check the worker's repo for uncommitted changes, untracked files that git doesn't ignore, and commits that are not on the remote. The check may resume a suspended worker.
  - If there is nothing to save, teardown proceeds.
  - **Age-based teardown:** the harness commits any uncommitted work as a WIP commit, pushes everything to a salvage branch on the remote (for example `klank/salvage/<effort>`), and then tears down. If the push fails, the worker is kept and I am notified (FR-60). The harness retries later (NFR-5).
  - **Manual teardown:** the harness refuses by default and shows what would be lost. I can choose to save to the salvage branch and tear down, or to discard. Discarding needs explicit confirmation in the CLI and in the command center.
  - Automatic teardown on a merged or closed PR (the opt-in policy in FR-18) skips the check, since the work is merged or deliberately abandoned. It still copies the session transcripts (FR-19).
  - Manual teardown of an effort whose PR is merged or closed still runs the check, since there may be work after the merge. It follows the manual rules above.
  - Build artifacts and other git-ignored files are not preserved.
- **FR-21** Per-worker vCPU, RAM and maximum disk size must be configurable, with a global default.

### 4.4 Identity and authentication

- **FR-22** Claude Code in workers must authenticate with a long-lived subscription token from `claude setup-token`, supplied via the `CLAUDE_CODE_OAUTH_TOKEN` environment variable.
- **FR-23** My interactive Claude login (`~/.claude/.credentials.json`) must **not** be copied into the master. Many VMs refreshing one shared login is avoided by design.
- **FR-24** Claude Code in workers must load my normal configuration: settings, CLAUDE.md, skills and MCP servers.
  - User-scoped MCP servers, onboarding state and per-project workspace trust live in `~/.claude.json`, not in `~/.claude/settings.json`. The master must therefore carry a `~/.claude.json` built for workers. It must not be a verbatim copy of the host's file.
  - It must include my user-scoped MCP server definitions and mark onboarding complete and the repo's path as trusted.
  - It must leave out host login and account identity (FR-23), and host-specific per-project state and history. Planners must check the exact fields against the pinned Claude Code version.
  - MCP servers that can't work in a worker, such as ones needing interactive OAuth or pointing at host-local services, must be either made to work or excluded. The refresh must report which ones were excluded and why. A silently missing server is not acceptable.
  - Note: Claude Code's "bare" mode skips this configuration and does not read `CLAUDE_CODE_OAUTH_TOKEN`, so it is not suitable here.
  - Headless operation must not be broken by first-run onboarding or workspace-trust prompts. Every master version must pass a smoke test before it becomes current, without touching existing workers. The test runs a trivial headless turn and confirms that settings, skills and MCP servers loaded, that no interactive prompt appeared, that `gh` is authenticated, and that git can reach the repo's remote with those credentials (for example `git ls-remote`). A failed smoke test fails the refresh (FR-10). Consultants are checked as part of the same smoke test, but a consultant that fails is left out of the version rather than failing the refresh (FR-46).
- **FR-25** `gh` and git push must work in workers with my identity, with no prompts, for the driver. Consultants must not have them (FR-50).
  - The credential comes from the token the refresh read from my host `gh` login (FR-9), not from a copy of `~/.config/gh`.
  - Where the token lives is left to planners: baked into the master, or supplied to each worker at spawn. Supplying it at spawn keeps it out of the image and lets it rotate without a refresh. Either way it falls under the credential risk in §6.

### 4.5 Running turns

- **FR-26** The harness must run turns non-interactively, in the worker's repo directory, with the prompt supplied by the harness.
- **FR-27** Turns must never block waiting for a permission approval.
  - **v1 policy:** the driver runs with permission checks bypassed. The disposable VM and trusted inputs (§1.1) are the safety boundary.
  - Consults run in the consultant's read-only mode instead (FR-50), and must not block either.
  - The policy must be a configuration setting, so a restricted allowlist can replace it later without code changes.
  - Any permission denial that does occur must be recorded and surfaced.
- **FR-28** The first driver turn of an effort must establish the driver's session. Every later driver turn for that effort must continue the same session. Consultant sessions are covered by FR-48.
- **FR-29** The full structured output of every turn, driver or consult, must be saved on the host, per effort and per turn, with the agent that ran it. The final outcome must be stored alongside it, including result text, exit status, cost (in whatever unit the agent reports) and any permission denials. Output must be written to the host as the turn produces it, not only when the turn ends, so a turn that is interrupted (NFR-8) or killed still leaves its output so far (FR-55).
- **FR-30** Only one turn, driver or consult, may run per worker at a time. A second turn for a busy worker must be queued. Driver turns and consults share one queue, in the order they were sent.
- **FR-31** I must be able to SSH into any worker and continue its session interactively in Claude Code. While I'm attached, the harness must not start turns on that worker.
  - I count as attached whenever an interactive Claude Code session is live on the worker, however I reached it: through a harness-provided attach command or a plain `ssh`. How this is detected is left to planners (§7.1).
  - While I'm attached, the effort has the attached hold (FR-38). A turn sent meanwhile is queued, not dropped (FR-30), and starts after I detach.
  - An open shell with no interactive Claude Code session does not count as attached.
  - If the chosen mechanism leaves a window in which a turn can start just as I attach, that window must be documented.
- **FR-32** A turn exceeding a wall-clock limit or maximum turn count, each configurable per agent, must be interrupted gracefully, so the session stays resumable, before any forced kill. Spend is limited by these and by the concurrency cap (NFR-1), not by dollar amounts.
- **FR-33** I must be able to abort a running turn from the CLI and from the command center, and to drop turns that are queued for an effort.
  - An abort uses the same graceful interrupt as FR-32, so the session stays resumable. A forced kill happens only if the turn hasn't stopped within a configurable grace period.
  - After an abort the effort is paused (FR-38). Queued turns are kept, and nothing starts until I resume the effort or send the driver a prompt. Starting a consult does not resume it. Sending the driver a prompt counts as a harness-initiated prompt for the circuit breaker (FR-42).
  - The turn's outcome is recorded as aborted, distinct from a crash or a limit being hit. Work already done in the worker is left as it is.
- **FR-34** If a turn fails because an agent's usage limit is reached, the harness must mark that agent rate-limited on every affected effort, queue that agent's pending turns, and retry after the limit resets. A limit on one agent does not hold another agent's turns. For example, Claude being rate-limited does not stop a consult with Codex. The command center and notifications must show this hold (FR-60).

### 4.6 Effort intake

- **FR-35** I must be able to start an effort from a GitHub issue reference. The issue's repo must be a target repo with a master (FR-4). Otherwise the effort is refused, with the reason. The harness:
  1. Spawns a worker.
  2. Builds the first prompt from the issue title and body plus a configurable instruction template.
  3. Runs the first turn.

  Efforts start **discuss-first**. The default template tells the driver to read the issue, explore the code, and tell me what it thinks and what it would propose. It must not push, open a PR or post anything to GitHub until I tell it to. The template also carries the conventions the driver follows when I do ask for a PR: branch naming, open it as a draft, link the issue. It also says what the harness has already done to the environment (FR-16), so the driver doesn't repeat it.
  - There is no separate autonomous mode. If I want a PR without reviewing first, I say so to the driver, either in the start command's optional extra instruction or in a later prompt.
  - Discuss-first is an instruction to the driver, not a technical guard (§6).
- **FR-36** I must be able to start an effort from a free-form prompt, for non-issue threads. I name the target repo, and the same refusal applies as in FR-35. It uses the same instruction template, so it is also discuss-first.
- **FR-37** The harness must keep a persistent registry of efforts: name, repo, issue (if any), PR number (none until a PR exists, FR-39), the driver's session identifier, each consultant's current session identifier (FR-48), master version, status (FR-38) and last activity.
- **FR-38** An effort's status has three parts, stored in the registry: its lifecycle state, its worker's condition, and any holds on starting turns. Every change to any part must be recorded as an event with a timestamp and a cause. The events feed live updates (§2.1), notifications (FR-60) and reconciliation (NFR-8).

  **Lifecycle state.** Every effort is in exactly one of these:

  | State | Meaning |
  |---|---|
  | spawning | Worker is being created, booted, brought up to date and its environment prepared (FR-12, FR-15, FR-16). |
  | active | The effort has a worker and takes turns, subject to holds. |
  | reviving | A new worker is being built from the effort's branch and its environment prepared (FR-19, FR-16). |
  | tearing down | Worker is being removed (FR-18, FR-20). |
  | torn down | No worker. The effort can be revived (FR-19): its PR is still open, or it has no PR (FR-39). |
  | done | The PR was merged or closed, or I marked the effort done (FR-39), and the worker has been removed (FR-43). |
  | failed | Spawn or revive failed, a prep step failed (FR-16), or the worker is unrecoverable. No turn starts. Needs my action (below). |

  **Worker condition.** For an effort with a worker: whether the worker is up or suspended (FR-17), and whether a turn is running, queued, or neither. Torn down and done efforts have no worker.

  **Holds.** Each hold stops new turns from starting. Holds are independent of each other and of whether the worker is up or suspended, so more than one can apply at once.

  | Hold | Set by | Cleared by |
  |---|---|---|
  | attached | I attach interactively (FR-31). | I detach. |
  | paused | The circuit breaker (FR-42), an abort (FR-33), a consult that changed the working tree (FR-50), or me. | I resume the effort or send the driver a prompt. Starting a consult does not clear it. |
  | rate-limited (per agent) | A turn hits that agent's usage limit (FR-34). | That agent's limit resets. |

  - A turn starts only when the effort is `active`, no hold applies to it, and no other turn is running on the worker (FR-30). Attached and paused hold every agent's turns. Rate-limited holds only the turns of the agent that hit the limit. A suspended worker is resumed first (FR-17).
  - Holds are cleared when the worker is removed.
  - The outcome of each turn, with the agent that ran it, is recorded separately from the status: succeeded, failed, interrupted (NFR-8), aborted by me (FR-33), stopped at the time or turn limit (FR-32), blocked by the usage limit (FR-34), or needs input (FR-53).
  - Every hold that applies must be visible (FR-52). Which single status the effort list leads with when several parts apply, for example attached while rate-limited, is left to planners.
  - The end-of-life and failure transitions (teardown, revive, done, failed) are fixed by the tables below. Planners must define the remaining changes, such as how holds and the worker's condition change while an effort is `active`. A transition that isn't defined is rejected, and reconciliation (NFR-8) resolves any mismatch between registry and reality through defined transitions only.

  **Failed.** An effort goes to `failed` when its spawn or revive fails, when a prep step fails (FR-16), or when its worker becomes unrecoverable. The cause and any log are recorded as an event, and an attention item is raised (FR-53). While the effort is `failed`:
  - No turn starts. A prompt or a consult is refused, with the reason.
  - New feedback on its PR raises an attention item under either feedback policy, and starts no turn (FR-41).
  - A merged or closed PR is recorded as usual (FR-43). If the merge/close teardown policy is on, the effort is torn down and becomes `done`.
  - If the worker is reachable, I can `ssh` into it to investigate (FR-13). It is suspended when idle like any worker (FR-17). Age-based teardown does not apply, so a failed effort waits for me.

  I leave `failed` in one of three ways, from the CLI or the command center:
  - **Retry.** The effort returns to the state it failed in (`spawning`, `reviving` or `active`) and resumes from the step that failed.
    - If the worker is up, the failed step and the steps after it are re-run on that worker.
    - Otherwise the partly built worker is removed and the spawn or revive starts again on a new one (NFR-5).
    - A first prompt or seed prompt that never ran is still queued, and runs once the effort is `active`.
    - If the retry fails, the effort is `failed` again. Retry is not offered when the worker is unrecoverable.
  - **Tear down.** This is manual teardown, with the work-preservation check (FR-20). If the worker can't be checked, because it is unrecoverable or never finished booting, the harness says so. Teardown then needs the same explicit confirmation as a discard. The effort ends `torn down` or `done` according to its PR, as for any manual teardown, and a `torn down` effort can be revived (FR-19).
  - **Mark done.** The effort is torn down first under the rules for tearing down, then becomes `done` (FR-39).

  **End-of-life and failure transitions.** These tables gather rules stated in FR-16, FR-18, FR-19, FR-20, FR-39, FR-41, FR-43 and FR-47, and the failed rules above, in one place. If they ever disagree with those requirements, the requirements govern and the tables are corrected.

  | From | Trigger | To |
  |---|---|---|
  | spawning, active, reviving or failed | Teardown starts: manual, age policy (not from `failed`), merge/close policy, or mark done (the scenarios below say which checks apply) | tearing down |
  | spawning | Spawn failed, or a prep step failed (FR-16) | failed |
  | active | A prep step failed (FR-16), or the worker is unrecoverable | failed |
  | tearing down | Completed; PR is open or there is none; not marked done | torn down |
  | tearing down | Completed; PR is merged or closed, or teardown was started by mark done | done |
  | tearing down | Salvage push failed (FR-20). The worker is kept and an attention item is raised. The harness retries (NFR-5). | the state it was in before |
  | torn down | I send the driver a prompt, or use "discuss feedback"; or new feedback on the open PR under the auto policy (FR-19) | reviving |
  | torn down | I mark it done (FR-39) | done |
  | torn down | I start a consult. It is refused, since there is no worker. I revive the effort by sending the driver a prompt first (FR-47). | (none) |
  | torn down | PR merged or closed | done |
  | reviving | New worker is up; the seed prompt is queued as its first turn | active |
  | reviving | Revive failed, or a prep step failed (FR-16) | failed |
  | failed | I retry. Not offered when the worker is unrecoverable. | the state it failed in: spawning, reviving or active |
  | failed | I send a prompt or start a consult. It is refused, with the reason. | (none) |
  | failed | New feedback on its PR, under either policy. An attention item is raised and no turn starts. | (none) |
  | done | Anything: a prompt or consult is refused and new feedback is ignored. `done` is final. | (none) |

  **Teardown scenarios.** "Check" is the work-preservation check of FR-20: uncommitted changes, untracked files git doesn't ignore, and commits not on the remote. Every teardown that reaches `torn down` or `done` first copies the session transcripts to the host (FR-19).

  | Trigger | PR | Check | Result |
  |---|---|---|---|
  | Manual | none or open | clean | Tear down, then `torn down`. |
  | Manual | none or open | dirty | Refuse and list what would be lost. I choose save to the salvage branch then tear down, or discard (explicit confirmation). Then `torn down`. |
  | Manual | merged or closed | as above | Same as the two rows above, because there may be work after the merge. Then `done`. |
  | Age policy (opt-in) | none or open | clean | Tear down, then `torn down`. |
  | Age policy (opt-in) | none or open | dirty | WIP commit, push to the salvage branch, tear down, then `torn down`. If the push fails, the worker is kept, I am notified and the harness retries. |
  | Age policy (opt-in) | merged or closed | as above | Same as the two rows above, then `done`. |
  | Merge/close policy (opt-in) | merged or closed | skipped | Tear down, then `done`. |
  | Merge/close policy off | merged or closed | none | Nothing is torn down. The merge or close is recorded and an attention item is raised (FR-43). |
  | Manual on a `failed` effort | any | as manual, if the worker can be checked | Same as the manual rows above. |
  | Manual on a `failed` effort | any | impossible: worker unrecoverable or never booted | The harness says the check can't run. Tear down only after explicit confirmation, as for a discard. Then `torn down`, or `done` if the PR is merged or closed. |
  | Mark done on an `active` or `failed` effort | any | as manual | Tear down first under the manual rules, then `done`. |
  | Mark done on a `torn down` effort | any | none | `done`. |

  **Revive scenarios.** A revive builds a new worker from the repo's current master with a fresh session. It checks out the salvage branch if one exists, otherwise the PR's remote branch, otherwise the default branch (FR-19). Consultant sessions start fresh too (FR-48). A consult never revives an effort (FR-47).

  | Effort | Event | Feedback policy | Result |
  |---|---|---|---|
  | `torn down`, PR open | New feedback | notify | Attention item only. No worker is created until I send the driver a prompt or use "discuss feedback". |
  | `torn down`, PR open | New feedback | auto | Revives by itself. |
  | `torn down`, PR open | I send the driver a prompt or use "discuss feedback" | any | Revives. |
  | `torn down`, no PR | I send the driver a prompt | n/a | Revives. There is no feedback to trigger it (FR-39). |
  | `done` | Anything | any | No revive. Prompts and consults are refused and feedback is ignored. |
- **FR-39** An effort may have no PR. This is a condition of the effort, not a separate kind of effort, and applies to free-form efforts (FR-36) and to issue efforts where the driver hasn't opened a PR yet or never does.
  - **Association.** The harness must notice when a PR for the effort comes to exist, and record it in the registry as an event. Planners choose the mechanism (§7.1). I can also link a PR by hand from the CLI and the command center, for a PR opened some other way. From then on the effort is treated as a PR effort in every respect.
  - **No PR means no feedback loop.** FR-40 to FR-43 apply only once a PR is associated. There is no feedback to handle until then, so no turn starts except from me. Looking for the PR itself is allowed, per the watching principle in FR-40.
  - **Ending.** There is no merge or close event, so only manual teardown ends the worker, or the opt-in age cutoff (FR-18) if I have turned it on, both subject to FR-20. Either leaves the effort `torn down`, which can be revived by hand (FR-19).
  - **Normal case.** Every effort starts with no PR (FR-35), and stays that way until I ask the driver for one.
  - **Mark done.** I must be able to mark a `torn down` effort, with or without a PR, as `done` from the CLI and the command center. This ends the effort for good. It can no longer be revived, and new feedback on its PR no longer revives it. I can also mark an `active` effort done, which tears it down first, subject to FR-20.
  - **Visibility.** The effort list (FR-52) shows "no PR" in place of the PR link. A turn that finishes on an effort with no PR raises the "waiting on me" attention item (FR-53).

### 4.7 PR feedback loop

- **FR-40** The harness must watch each effort's open PR for its state (ready for review, merged, closed), review comments, review summaries, conversation comments and failing CI checks. Each item must be handled exactly once. Nothing may be missed or handled twice, including items that arrive during a turn. An item counts as handled once it has been included in a prompt sent to the driver, or when I dismiss it.
  - **Watching principle.** The harness watches GitHub whenever it has a reason to, at a pace suited to that reason: whether a PR exists for an effort's branch (FR-39), PR state, new feedback, CI results. The harness does the watching, not the agents. The driver sees a result only in a prompt, and only when the feedback policy (FR-41) calls for a turn, so watching a PR costs no tokens. How quickly a change is noticed is configurable, with defaults chosen by planners. API use should be proportionate, for example a slower pace for quiet or suspended efforts and a faster one just after a push. The driver may still check GitHub itself during a turn. Consultants can't (FR-50).
- **FR-41** What happens when new feedback appears is set by a **feedback policy**, configurable globally and per repo, with a per-effort override.
  - **notify (default).** No turn starts. The harness raises an attention item (FR-53) naming the PR, the authors and the number of new items. Items that arrive close together are batched by a configurable quiet period, and the existing item is updated rather than duplicated. The command center offers a "discuss feedback" action. It sends the effort a prompt, built from a configurable template, that includes the new items and asks what the driver thinks of each and what it would do, without making changes. I can also write my own prompt instead, for example "rob left feedback on PR 123, what do you think?"
  - **auto.** The harness continues the effort's session at once with a prompt saying there is new feedback on PR #n to read and address. A submitted review triggers a turn immediately. Loose comments trigger one after the quiet period, so a burst of comments produces one turn.
  - The author allowlist (FR-44) applies under both policies.
- **FR-42** The harness must never start a turn, or raise an attention item, in response to something the driver itself posted. Because the driver posts as my GitHub account, this is decided by a marker on agent-authored content, not by author. Consultants have no GitHub access (FR-50), so only the driver's posts need marking.
  - Everything the driver posts to GitHub (comments, review replies, the PR body) must carry a marker, such as a hidden HTML comment identifying the effort.
  - The harness must apply the marker itself. It must not depend on the driver remembering to add it. The mechanism is left to planners (§7.1).
  - Every route the driver has for writing to GitHub from a worker must be covered, not only `gh`. For example, a GitHub MCP server carried over under FR-24 must either be covered or be excluded and reported as FR-24 describes. Direct API calls with the token, such as with `curl`, can't be fully prevented. This is a known limit, and the circuit breaker below is the backstop.
  - Feedback without the marker is handled according to the feedback policy (FR-41), including comments and reviews from my own account. This lets me steer an effort from the PR as well as through the harness (FR-59), though the harness is the main way.
  - Failing CI checks have no author and are handled by the feedback policy too (FR-40).
  - **Circuit breaker (auto policy only):** after a configurable number of consecutive feedback-triggered turns on one effort (default 5) with no harness-initiated prompt in between, the harness must pause the effort and notify me (FR-60). This bounds the damage if a marker is ever missed. Consults neither count towards the breaker nor reset it.
- **FR-43** When a PR is merged or closed, the harness must record that on the effort, stop watching the PR for feedback, and raise an attention item (FR-53) saying the effort is ready to tear down. If the merge/close teardown policy (FR-18) is on, it also tears the worker down. Once the worker of such an effort is removed, whether by that policy or by me, the effort is `done`.
- **FR-44** An optional **feedback author allowlist** is a prompt-injection mitigation (§6). It is unrelated to loop prevention (FR-42).
  - It is off by default. With no allowlist configured, unmarked feedback from any author is handled under the feedback policy (FR-41), since all authors are trusted in v1 (§1.1).
  - When configured, feedback from authors not on the list must neither trigger a turn nor be included in any prompt.
  - It can be set globally or per repo, and must be changeable without code changes.

### 4.8 Consultant agents

Every effort has one driver, Claude Code, which does the work. A consultant is another coding agent that I bring into an effort when I want one, usually to review the driver's changes or for a second opinion. Consultants read and advise. They never change the code, commit, push or post, and only I start a consult.

- **FR-45** A consultant is a coding agent configured globally, by name (for example `codex`). Any effort can consult it. There is no separate step to add it to an effort.
  - An agent can be a consultant only if its CLI provides everything the harness relies on:
    1. A non-interactive mode, run with a prompt supplied by the harness, that streams structured output (FR-29, FR-55).
    2. Sessions that can be continued by identifier across turns (FR-48).
    3. A credential that many workers can use at once, with no shared login that refreshes itself (FR-46).
    4. A read-only mode (FR-50).
    5. A graceful interrupt that leaves the session resumable (FR-32, FR-33).
    6. A version that can be pinned with auto-update disabled (FR-8).

    Agents that fall short of this are not supported.
  - Codex is the first consultant. Planners must check it against this list for the pinned version. Its credential is an open question (§7).
  - Adding another consultant that meets the list must not need changes outside the part of the harness specific to that agent.
- **FR-46** Each consultant must be installed, authenticated and configured in every master.
  - **CLI.** Pinned, with auto-update disabled (FR-8).
  - **Credential.** A credential that many workers can use at once, such as an API key or a long-lived token. My interactive login on the host is never copied, for the reason in FR-23. Where the credential lives is left to planners, as for the GitHub token (FR-25). It falls under the credential risk in §6.
  - **Configuration.** The refresh brings in the consultant's configuration from the host, from a configurable list per consultant (for Codex, for example, its config file and its global instructions file), leaving out host login state. MCP servers that can't work in a worker are made to work or excluded and reported, as in FR-24. Instructions are not kept in sync between agents. I maintain each agent's instructions myself.
  - **Smoke test.** The FR-24 smoke test runs a trivial consult for each consultant. It confirms that the consult ran headless with no interactive prompt, that the consultant's configuration loaded, and that it could not authenticate to GitHub (FR-50). A consultant that fails is left out of that master version. The refresh still succeeds and reports which consultant was left out and why.
  - A worker can consult only the consultants that passed on its own master version (FR-12). A consultant added later is not available to an existing worker. It becomes available when the effort is revived on a newer master.
- **FR-47** Only I start a consult: from the conversation view (FR-55), the CLI or the command center. The driver cannot start one, and no policy starts one.
  - **Asking.** I address a message to a consultant by name. A message not addressed to a consultant goes to the driver.
  - **Review.** A review action sends a consult built from a configurable review template. It asks the consultant to review the effort's changes against the base branch and say what it finds, without changing anything. I can add my own instruction to it.
  - **Turn rules.** A consult is a turn. It waits in the worker's queue (FR-30), is held by attached, paused and that consultant's rate-limited hold (FR-38), resumes a suspended worker (FR-17), and is subject to the limits, abort and drop of FR-32 and FR-33.
  - **Refusal.** A consult is refused, with the reason, unless the effort is `spawning`, `active` or `reviving` and the consultant is available on its master version (FR-46). While the effort is spawning or reviving, the consult is queued behind the driver's first turn. A consult never revives a `torn down` effort. I revive it by sending the driver a prompt (FR-19).
- **FR-48** Each consultant has its own session per effort, separate from the driver's and from other consultants'. The first consult establishes it, and later consults continue it.
  - **Clearing.** I can clear a consultant's context on an effort from the conversation view, the CLI and the command center. Its session ends, and the next consult starts a new one with the full briefing (FR-49). The old transcript stays on the host. A clear made while a consult is running applies after it ends. Clearing costs no turn. The conversation view shows where the context was cleared.
  - Consultant sessions end when the worker is removed (FR-19). After a revival, the next consult starts a new session.
  - Interactive access to a consultant's session (FR-31) is not required.
- **FR-49** The harness must build each consult's prompt from a configurable consult template. The consultant gets what it needs without the driver's whole conversation.
  - Every consult includes my request.
  - The first consult of a consultant session also includes:
    - that it is a read-only consultant on this effort, with no GitHub access (FR-50)
    - the repo, the effort's branch and its base branch
    - the PR number, title and body, if there is a PR
    - what the harness did to the environment (FR-16), so it uses the running app rather than starting its own
  - Every consult also includes the conversation between me and the driver since this consultant's last consult. For a new consultant session, that is everything since the start of the driver's current session. This covers my messages, other prompts sent to the driver (such as feedback) and the driver's text. It does not cover the driver's tool calls or their output. The consultant reads the code itself.
  - If that conversation is too long, planners choose how to shorten it, and the prompt says that it was shortened.
  - Externally authored text passes through the content-filter seam (NFR-6) and the author allowlist (FR-44), as for any prompt.
- **FR-50** A consult must not change anything. The harness enforces this, not only the consultant's instructions.
  - **No GitHub access.** A consult runs without my GitHub credential. `gh` is not authenticated for it, and git cannot push or fetch with my identity. It works from the worker's local clones.
  - **Read-only mode.** A consult runs in the consultant's read-only mode (FR-45).
  - **Tree check.** The harness records the state of the worker's repo before and after each consult: HEAD, branches, the index, and tracked and untracked files that git doesn't ignore. If anything changed, the harness:
    - records an event
    - pauses the effort (FR-38), so the driver doesn't build on the change unseen
    - raises an attention item (FR-53) showing what changed

    I can then revert the change or keep it, from the CLI and the command center, and resume.
  - Git-ignored files, such as build output, and data outside the repos, such as the app's database, are not checked.
  - **Known limit.** A consultant that reads a credential from the worker's disk and uses it directly can't be fully prevented, as with direct API calls in FR-42. Anything it posts this way carries no marker. Under the auto policy, the circuit breaker (FR-42) is the backstop.
- **FR-51** The driver must see what consultants said. The prompt of the next driver turn includes the reply of every consult that finished since the previous driver turn, each labelled with the consultant and my request. A consult never starts a driver turn by itself.

### 4.9 Visibility

- **FR-52** I must be able to list all efforts with:
  - status (FR-38): lifecycle state, whether the worker is up or suspended, whether a turn is running or queued, and any holds
  - PR link, or "no PR" (FR-39)
  - last turn time and outcome, and which agent ran it
  - cumulative cost, per agent
  - the master version its worker is pinned to, and the disk space its worker uses (FR-56)
  - the app's URL and whether it is up (FR-16)
- **FR-53** The harness must raise an **attention item** for anything that needs me. Attention items are stored with the harness, not in a client, so they persist when the command center is closed (§2.1).
  - Each item has an effort, a type, a timestamp and a status (open or acknowledged). I can acknowledge one explicitly. Resolving the cause, such as resuming a paused effort, closes it.
  - Types:
    - turn needs input: the driver or a consultant says it is blocked on me
    - turn failed, or stopped at a time or turn limit (FR-32)
    - permission denial (FR-27)
    - an agent rate-limited on the effort (FR-34), the effort paused (FR-42, FR-33) or failed (FR-38)
    - a consult changed the working tree (FR-50)
    - teardown blocked because the salvage push failed (FR-20)
    - the effort's branch changed build settings that its master can't pick up (FR-4)
    - PR opened or marked ready for review
    - new feedback on a PR, under the notify policy (FR-41)
    - PR merged or closed, so the effort is ready to tear down (FR-43)
    - effort is waiting on me: a turn finished and nothing else is queued
  - **Needs input:** the harness must detect a turn, driver or consult, that ends with the agent blocked on a decision from me. Planners choose the mechanism for each agent (§7.1). A turn that ends this way is recorded with the outcome "needs input" and is not treated as an ordinary success.
  - Which types produce a desktop notification (FR-60) is configurable. By default, all of them do.
  - Open items must show in the effort list (FR-52).
- **FR-54** I must be able to view a readable rendering of any turn, finished or running: assistant text and tool calls. FR-55 presents these as a continuous conversation.
- **FR-55** Each effort must have a **conversation view**, in the command center and with CLI equivalents (for example follow and send, FR-63). This is the main way I talk to the driver and to consultants.
  - It shows the whole conversation across turns, in order: my messages, and each agent's text and tool calls in the readable rendering of FR-54. Each message I send is a turn underneath (FR-26), which the view need not expose.
  - Every message shows which agent it is from or to. Consults appear in the same conversation as the driver's turns, in order. The view also shows where a consultant's context was cleared (FR-48).
  - I choose which agent each message goes to. The driver is the default (FR-47).
  - A running turn streams into the view. Text and tool calls must appear within a few seconds of being produced.
  - A client that connects, or reconnects, at any time, including mid-turn, must receive the conversation so far and then continue live, with nothing missing or duplicated. Any number of clients may view the same effort.
  - I can send a message from the view. If a turn is running, the message is queued (FR-30), shown as queued, and can be dropped (FR-33). Interrupting a running turn with a message is not required.
  - Each prompt records which agent it was sent to and where it came from: me in the command center, me in the CLI, "discuss feedback" (FR-41), the review action (FR-47), or the auto feedback policy.
  - **Answering a question.** An attention item of type "needs input" (FR-53) opens the conversation at the agent's question. Replying in the view sends the reply to the agent that asked, driver or consultant, and closes the item.
  - Viewing does not attach (FR-31) and has no effect on the effort's status.
  - The view matches the host-side record (FR-29) and survives a harness restart (NFR-8).
- **FR-56** I must be able to see where disk space goes and what each worker is pinned to, from the CLI and the command center.
  - **Per worker** (suspended ones included): the master version it is pinned to and whether that is the current master, the disk space its clone actually uses on the host, and the configured maximum (FR-21).
  - **Per master version:** its repo, whether it is current for that repo, how many workers depend on it, and its size on disk.
  - **Totals** for worker clones and for master versions.
  - Efforts without a worker (`torn down`, `done`) are listed without disk figures.
  - Pruning an unreferenced master version (FR-11) and tearing down a worker (FR-18, FR-20) can be done from this view.
  - "Actually uses" means space allocated on the host for the clone's changes, not the virtual size of the disk (NFR-3).

### 4.10 Command center (intent only; detailed requirements to follow)

A native desktop application that is my main way of seeing and steering efforts. This section records intent so planners can make architecture decisions that fit it. It is not a complete specification.

- **FR-57** The command center must show all efforts and their live status, covering everything in FR-52, and update without manual refresh.
- **FR-58** It must let me drill into an effort: turn history, the conversation view (FR-55), PR link, cost per agent, and disk use and master pinning (FR-56).
- **FR-59** It must let me perform the lifecycle operations available in the CLI: start an effort (issue or free-form), abort a turn or drop queued turns (FR-33), suspend, resume, tear down, mark done (FR-39), use the conversation view (FR-55), consult, request a review and clear a consultant's context (FR-47, FR-48), revert or keep a consult's changes (FR-50), see disk use and pinning (FR-56), restart the app or re-run prep (FR-16), and manage master versions. Sending a follow-up prompt to an existing effort is required. It is my main way of talking to the driver (FR-55), and GitHub is secondary. The full set of operations is whatever the CLI offers (FR-63).
- **FR-60** It must surface the attention items defined in FR-53, through desktop notifications and in the effort list. Open items raised while the command center was closed must still be shown when it starts.
- **FR-61** It must provide a way to open an interactive session on a worker (FR-31) by launching an external terminal.
- **FR-62** It must provide a way to open an effort's app (FR-16) in the host's default browser.
- **FR-63** It must be a client of the harness interface in §2.1. It holds no effort state of its own and adds no capability the CLI lacks.

Platform and technology constraints (see also NFR-7):
- It must be built with a native toolkit (such as GTK or Qt). Electron is excluded. A lightweight webview-based shell (such as Tauri) is allowed only as a fallback, if planners show that a native toolkit is a poor fit for a specific requirement such as the turn rendering in FR-54. The choice of toolkit is left to planners, and may influence the harness core's language.
- In v1 it runs on the same machine as the VM host. The harness interface should not make a remote client impossible later.
- It must run well on Wayland (Hyprland on Omarchy) and on Fedora's default desktop. It must not depend on a system tray, which Hyprland does not provide by default.
- It is not a TUI.

## 5. Non-Functional Requirements

- **NFR-1 Concurrency:** support at least 3 simultaneous workers on one host, with the cap configurable and defaulting to 3. Per-worker resource defaults are set during planning, sized so 3 fit comfortably on the host.
- **NFR-2 Spawn time:** a new worker should be SSH-ready within about 30 s of the spawn command. This is a target, not a pass/fail gate, until it has been measured (§7). The first-boot fetch (FR-15) is not included.
- **NFR-3 Disk:** a worker's disk clone should start near zero size and grow only with the effort's changes, up to a configurable maximum.
- **NFR-4 Master safety:** nothing may write to a master version that has dependent workers.
- **NFR-5 Idempotence:** every operation must be safe to retry after a failure, without corrupting state or leaking VMs.
- **NFR-6 Content-filter seam:** all externally authored text (issue bodies, PR comments, review text) must pass through a single, replaceable processing stage before it reaches a prompt. In v1 that stage does nothing. A comparable seam should exist for worker network egress. Both exist so scrubbing can be added later without re-architecting.
- **NFR-7 Host portability:** the harness and command center must install and run on both Omarchy (Arch-based) and Fedora. Packaging may differ per distribution. Dependencies must be available on both, and nothing may assume a particular distribution's paths or package manager at runtime.
- **NFR-8 Reconciliation:** the harness must survive a restart of itself or a reboot of the host without losing efforts. On start, and periodically, it must compare the registry with the VMs, disks and clones that actually exist, then adopt, resume or clean up anything that disagrees. A turn that was running when the harness died is marked interrupted and remains resumable, unless it is re-attached (below). Orphaned VMs and disks are removed.
  - **In-VM turn processes.** The harness dying may leave the turn's agent process (`claude`, or a consultant's CLI) running on the worker. No second turn process may run alongside it. Before the effort can accept another turn (FR-30), reconciliation must either re-attach to the surviving process or stop it, gracefully first as in FR-32. No effort may start a turn, or show no turn running, while a turn process the harness is not tracking is still running on its worker.
  - **Complete the record.** Before marking the turn interrupted, reconciliation must bring the host-side record (FR-29) up to date with everything the turn produced, so the conversation view (FR-55) does not lose the end of the turn.
  - **Not required in v1:** re-attaching to a surviving turn so that it completes. Losing the rest of an in-flight turn on a harness restart is accepted. Planners may re-attach, if they can still meet every other requirement here.
  - Whether the turn process survives the harness losing its connection to the worker must be checked against the pinned version of each agent's CLI and the chosen transport, not assumed.

## 6. Accepted Risks

v1 deliberately "runs with scissors": private repos only, trusted authors only, permission checks bypassed. The risks below are accepted on that basis. If any premise in §1.1 changes (public repos, outside contributors), these must be revisited first.

- **Full credentials in every worker.** Each worker can push to any repo and act as me on GitHub. This is accepted for v1.
  - Possible mitigation: use a fine-grained GitHub token limited to the target repos.
  - Each consultant's credential (FR-46) is in every worker too.
- **Prompt injection via issue and PR text.** An agent reading text it treats as instructions runs with my credentials and without permission prompts. This is accepted because all authors are trusted.
  - v1 mitigation, off by default: the optional feedback author allowlist (FR-44).
  - Future mitigation: scrubbing through the NFR-6 seam and labeled-issue intake.
- **Discuss-first is an instruction, not a guard.** Nothing technical stops the driver from pushing or posting to GitHub before I say so. The instruction template (FR-35) is the only control in v1.
  - Possible later guard: have the marking mechanism (FR-42) hold outbound posts until I release them.
- **A consultant's read-only limit is not absolute.** The harness keeps my GitHub credential out of a consult, runs it in read-only mode and checks the working tree afterwards (FR-50). A consultant that finds a credential on the worker's disk and uses it directly can't be fully prevented. Changes outside the repos, such as to the app's database, are not detected.
- **No egress filtering in v1.** Workers have the same network access as my machine. The NFR-6 seam leaves room to add it.

## 7. Open Questions

Questions that need an answer from me:

1. **Command center detail.** §4.10 records intent only. Detailed requirements (screens, interactions, the turn rendering) still have to be written. They can follow the harness requirements, but §2.1 must hold in the meantime.
2. **Spawn time target (NFR-2).** Is 30 s the right target, or should it be a hard limit? It should be settled once real spawn times are measured.
3. **Codex credential (FR-46).** Use an API key, which is billed per use outside my subscription with no dollar cap (FR-32), or a long-lived subscription token, if Codex offers one that meets FR-46. Codex's interactive login refreshes itself and can't be shared across workers.
4. **First guest distribution (FR-6).** Which distribution v1 supports first.

### 7.1 Deferred to planners

These are decided during planning. The requirements set the constraints and leave the choice open.

- Command center toolkit, and whether it affects the core's language (§4.10).
- Per-worker resource defaults, sized so 3 workers fit comfortably on the host (NFR-1, FR-21).
- Exact `~/.claude.json` fields to carry into the master (FR-24).
- Where the GitHub token lives: in the master, or supplied to each worker at spawn (FR-25).
- The remaining transitions (FR-38), and which single status the effort list leads with when several parts apply.
- How a turn that ends blocked on me is detected, for each agent (FR-53).
- Whether the content-filter seam (NFR-6) also enforces the author allowlist (FR-44).
- Each consultant's configuration list and credential location (FR-46).
- How a consult is kept from my GitHub credential, how the working tree is checked, and how a consult's change is reverted (FR-50).
- How the briefing is shortened when the conversation since the last consult is too long (FR-49).
- How attachment is detected (FR-31). Options include an attach command that holds a per-worker lock, a check for a live interactive `claude` process before each turn, and a `claude` wrapper in the worker that takes the same lock however it is launched. The wrapper closes the race that a check before each turn leaves open.
- How the marker is applied (FR-42), for example a `gh` wrapper in the worker, and how other routes for writing to GitHub are covered.
- How a PR is associated with an effort (FR-39). Suggestions: the marking mechanism (FR-42) reports PRs the driver creates, the harness looks up PRs for the effort's branch when a turn ends, and a slow poll is added if that proves useful.
- How worker names resolve on the host (FR-13), and how the app is kept running inside the worker (FR-16).
- The format and location of the harness configuration, and the format and path of the repo definition file (FR-3, FR-4).
- Which version files a repo definition can defer to for its toolchain (FR-4).
- How a full build is done: where base images come from and how the toolchain is installed (FR-9, FR-6).
- How host secrets are stored (FR-5).

### 7.2 Decisions

| Topic | Decision |
|---|---|
| Loop prevention | Mark everything the driver posts and ignore marked content. Unmarked feedback, including from my own account, is handled under the feedback policy. Under auto, a circuit breaker backstops a missed marker (FR-42). |
| Feedback author filtering | Optional allowlist, off by default, as a prompt-injection mitigation only (FR-44). |
| Permission policy | Broad autonomous action for the driver in v1, configurable. Consults run read-only (FR-27, FR-50). |
| Trigger model | Manual start only in v1. |
| Efforts with no PR | No PR is a condition, not a kind of effort. Every effort starts without one. No feedback loop until a PR exists. Ends by manual teardown, revived by hand, and can be marked done (FR-39). |
| Where I talk to agents | The command center's conversation view is the main channel, for the driver and consultants alike. GitHub is secondary (FR-55). |
| Starting an effort | Discuss-first. The driver opens a PR only when I tell it to. No separate autonomous mode (FR-35). |
| PR feedback | Notify by default. I ask the driver what it thinks before anything is changed. Auto is an opt-in policy (FR-41). |
| Teardown | Manual by default. Merge/close and age teardown are opt-in policies, off by default (FR-18). |
| Idle behavior and session continuity | Suspend on idle and keep workers until I tear them down. A torn-down effort can be revived from its branch with a fresh session (FR-17, FR-18, FR-19). |
| Concurrency | Default cap of 3 workers (NFR-1). On usage-limit errors, hold that agent's turns, queue them and retry after reset (FR-34). |
| Cost guardrails | Wall-clock and max-turn limits, per agent, plus the concurrency cap, with no dollar caps (FR-32). |
| Other agents | Claude Code is always the driver. Other agents join an effort only as read-only consultants, and only when I start a consult. No agent-to-agent automation in v1. Codex is the first consultant. Only agents whose CLI meets FR-45 are supported (FR-45, FR-47). |
| Consultant sessions | One per consultant per effort, kept across consults. I can clear it. The driver sees consult replies in its next prompt (FR-48, FR-51). |
| Audience | A personal tool: one user, on their own host (§1.1). |
| Repos and masters | One master per target repo. Its guest distribution and toolchain come from the repo's definition, which is committed in the repo and can be supplied or replaced by my harness configuration. Every effort belongs to one repo (FR-4, FR-8). |
| Guest distributions | Set per repo. v1 supports at least one. Adding another is confined to distribution-specific code (FR-6). |
| Configuration | Human-editable text files: my harness configuration, plus a definition per repo. Every setting is either a build setting or a runtime setting (FR-3). |
| Environment and app | The harness prepares the environment (migrate, seed) and starts the app, per repo, with no agent turn. Apps are reached by worker name. Localhost forwarding is out of scope, and OAuth repos use a dev bypass (FR-13, FR-16). |
| Master refresh | Manual in v1. A full build from the repo's definition when a build setting has changed, an incremental refresh otherwise. Both produce a new version (FR-9). |
| Command center toolkit | Native toolkit preferred; Electron excluded; a lightweight webview shell is an allowed fallback only if native proves a poor fit (§4.10). |
| Command center location | Same machine as the VM host in v1, without precluding a remote client (§4.10). |
| Interactive access from the GUI | Launch an external terminal (FR-61). |

## 8. Acceptance Criteria (v1)

Each criterion says what to do and what observable result passes it. "Observable" means something visible in the registry, the event log, the host-side turn records, CLI or command center output, the host's VM and disk listings, or GitHub. Nothing is judged by feel.

**Fixtures.** A private test repo with a small test suite (including a test that binds a fixed port) and a build step. The repo also has a prep step that migrates and seeds a database, and a small web app with a health endpoint, so the environment in FR-16 can be tested. Its repo definition (FR-4) is committed. A second private test repo with no committed definition. A second GitHub account, used as an "other author". Test hooks that let a test shorten the idle, quiet-period and age timers, stub a usage-limit failure for a given agent, make a spawn fail before the worker boots, disable marking (FR-42), let a consult write despite read-only mode, and run a command in a consult's environment (FR-50). Codex configured as a consultant (FR-45). Anything that depends on live model behaviour is also tested against canned turn output, so the harness logic is verified without relying on the model.

### 8.1 Setup and configuration

- [ ] **AC-1 First-time setup** (FR-1, FR-2, FR-5)
  - Do: on a clean supported host, follow the documented steps from installation to a first effort. Run the host check before providing the Claude token and again after.
  - Pass when: each step reports what it did and what comes next. The first host check reports the missing token and what to do about it. The second passes. Re-running any step changes nothing. The effort runs its first turn. The token appears in neither the harness configuration nor the repo definition.
- [ ] **AC-2 Invalid configuration** (FR-3, FR-6)
  - Do: with an effort active, (a) give a setting in the harness configuration an invalid value, and (b) supply a repo definition that names an unsupported distribution.
  - Pass when: each is rejected with an error naming the file and the setting. The harness carries on with the last valid configuration, and the effort's status is unchanged.
- [ ] **AC-3 Build and runtime settings** (FR-3, FR-9)
  - Do: with a worker running, change a runtime setting (the quiet period) and a build setting (a toolchain version in the repo definition, merged to the default branch). Refresh the repo's master and spawn a second worker. Then refresh again with no further change.
  - Pass when: the new quiet period applies to the next feedback without a new master. The repo shows its master as out of date before the refresh. The first refresh is a full build and says why. The second worker has the new toolchain version, and the first worker still has the old one and is still on its old master version. The second refresh is incremental.
- [ ] **AC-4 Repos, definitions and override** (FR-4, FR-8, FR-12, FR-35)
  - Do: (a) list the second test repo in the harness configuration and try to start an effort in it. (b) Supply its definition in the harness configuration, refresh its master, and start an effort. (c) Override the first test repo's committed definition with a different app port, refresh, and start an effort.
  - Pass when: in (a), the effort is refused because the repo has no master. In (b), the second repo gets its own master version, separate from the first repo's, and its worker contains only that repo. In (c), the app runs on the override's port, and the harness shows that an override is in effect.
- [ ] **AC-5 Definition changed on a branch** (FR-4, FR-16)
  - Do: on an effort's branch, (a) change a prep step in the repo definition and re-run prep. (b) Change a toolchain version and re-run prep.
  - Pass when: in (a), the changed prep step runs. In (b), an event is recorded and an attention item says that the branch changed build settings its master can't pick up. The worker's toolchain is unchanged.

### 8.2 Master and workers

- [ ] **AC-6 Refresh leaves workers alone** (FR-9, FR-11, NFR-4)
  - Do: with one worker running from master v1 and mid-turn, refresh the master.
  - Pass when: master v2 exists and is current. v1 still exists and is unchanged. The worker finishes its turn, and the registry still shows it on v1.
- [ ] **AC-7 Failed smoke test** (FR-10, FR-24)
  - Do: make the smoke test fail (for example, an invalid GitHub token), then refresh.
  - Pass when: the refresh reports failure with the reason. The current master has the same version and image checksum as before. No partial version appears in the version list or on disk.
- [ ] **AC-8 Worker identity** (FR-12, FR-13, FR-14)
  - Do: spawn two workers from the same master version.
  - Pass when: both are reachable as `ssh <worker-name>`. Hostnames, MAC addresses, SSH host keys and machine-ids all differ.
- [ ] **AC-9 Fresh code** (FR-15)
  - Do: push a new commit to the test repo's default branch after the master was built, then spawn a worker.
  - Pass when: before the first turn starts, the worker's repo is at that commit.
- [ ] **AC-10 Environment and app ready** (FR-13, FR-16)
  - Do: spawn a worker for the test repo, then read the event log. From the host, request the app's health endpoint at `http://<worker-name>:<port>`. Spawn a second worker for the same repo.
  - Pass when: the events show the services, prep steps and app start in order, all before the first turn starts. The seeded data is visible through the app. The health check passes from the host for both workers on the same port. The first prompt of the session names the app's URL and its logs. Spawning used no turn and no tokens.
- [ ] **AC-11 Credentials** (FR-22, FR-23, FR-25)
  - Do: in a new worker, check `gh auth status`, run a `git push` to the test repo, and look for `~/.claude/.credentials.json`.
  - Pass when: `gh` is authenticated, the push succeeds without any prompt, and the credentials file does not exist. A headless turn runs using `CLAUDE_CODE_OAUTH_TOKEN`.
- [ ] **AC-12 Idle suspend and resume** (FR-17)
  - Do: set a short idle period and leave an effort idle. Then send it a prompt.
  - Pass when: the effort's worker is suspended and its disk is kept. The prompt resumes the worker automatically and continues the same session ID.
- [ ] **AC-13 App persists and failure is visible** (FR-17, FR-16)
  - Do: (a) after AC-10's worker finishes a turn, request the health endpoint. (b) Suspend the worker, then send a prompt. (c) Kill the app process and restart it from the CLI. (d) Spawn a worker whose prep step fails.
  - Pass when: in (a), the app still responds. In (b), it responds again once the worker is resumed and before the turn starts. In (c), the restart records no turn. In (d), the effort is `failed`, an attention item is open, and the prep log is on the host. With the repo set to continue past failure, the effort proceeds and the failure is still recorded.
- [ ] **AC-14 Merge, teardown and pruning** (FR-11, FR-18, FR-43)
  - Do: (a) with the merge/close teardown policy off, merge an effort's PR. (b) Tear that effort down by hand. (c) Turn the policy on and merge a second effort's PR. (d) Try to prune the master version they used, first while another worker still uses that version and again once none does.
  - Pass when: in (a), within the configured detection time the effort records the merged PR and an attention item is open, while the worker, its disk and the effort's status are unchanged, and later comments on the merged PR raise nothing. In (b), the worker and disk are gone, the transcript is on the host and the state is `done`. In (c), within the configured detection time the worker and disk are gone, the state is `done` and the transcript is on the host. In (d), pruning is refused while a worker depends on the version and succeeds when none does.

### 8.3 Turns and sessions

- [ ] **AC-15 Turn records** (FR-26, FR-28, FR-29)
  - Do: run a first turn and a follow-up turn on an effort.
  - Pass when: the host has the full structured output for each turn, per effort and per turn. Each has an outcome record with result text, exit status, cost and permission denials, and names the agent that ran it. Both turns share one session ID.
- [ ] **AC-16 Turn limits** (FR-32)
  - Do: set a very short wall-clock limit and give the effort a long task. Then send a follow-up prompt.
  - Pass when: the turn is interrupted before any forced kill, its outcome is "stopped at limit", and the follow-up continues the same session.
- [ ] **AC-17 Abort** (FR-33)
  - Do: start a turn, queue a second one, then abort the first.
  - Pass when: the turn stops gracefully with outcome "aborted". The effort is `paused` and the second turn is still queued. Sending the driver a prompt starts a turn in the same session.
- [ ] **AC-18 Attach** (FR-31)
  - Do: (a) attach with the attach command. (b) Separately, `ssh` in directly and start an interactive `claude`. In each case, send the effort a prompt from the CLI while attached. Then detach.
  - Pass when: in both cases the effort shows `attached`, the turn is held and queued, and it starts after detach. The session transcript shows no interleaved writes.
- [ ] **AC-19 Rate limit** (FR-34)
  - Do: stub a usage-limit failure on a turn, with a reset time a short way ahead.
  - Pass when: the effort shows `rate-limited` and I am notified. Pending turns are queued. After the stubbed reset, the turns run with no action from me.
- [ ] **AC-20 Needs input** (FR-53, FR-60)
  - Do: (a) feed the harness canned turn output in which the agent is blocked on a decision from me. (b) Run one live turn on a task that cannot proceed without a decision from me. In both cases, keep the command center closed until after the turn ends.
  - Pass when: the outcome is "needs input", an attention item is open, and a notification was sent. When the command center starts, it shows the open item. Replying to the question in the conversation view starts a turn in the same session and closes the item.

### 8.4 Concurrency and the feedback loop

AC-22 to AC-26 run with the feedback policy set to **auto**. AC-41 covers the default **notify** policy.

- [ ] **AC-21 Three concurrent efforts** (FR-35, NFR-1)
  - Do: start three efforts from three issues at the same time. When each has finished its first turn, tell each to open a draft PR.
  - Pass when: the first-turn records show overlapping time windows for all three, and GitHub shows no PR, push or comment from any of them yet. After the instruction, each effort has its own worker and its own draft PR linked to its issue, and the follow-up turns also overlap. Each PR contains only its own effort's commits. The fixed-port test passes in all three.
- [ ] **AC-22 Feedback reaches the right session** (FR-40, FR-41)
  - Do: with three efforts in flight, have the other author submit a review on PR A.
  - Pass when: only effort A runs a new turn, in its existing session. A follow-up commit appears on PR A's branch. B and C have no new turns.
- [ ] **AC-23 Each item handled once** (FR-40, FR-41)
  - Do: post three loose comments in quick succession, then one more during the resulting turn.
  - Pass when: the first three produce one turn after the quiet period. The fourth is handled by a later turn. The turn records show no comment ID handled twice and none missed.
- [ ] **AC-24 Loop prevention** (FR-42)
  - Do: have the driver reply to a review comment. Then post an unmarked comment from my own account.
  - Pass when: the driver's reply carries the marker, and no turn starts from it within twice the quiet period. My unmarked comment starts a turn.
- [ ] **AC-25 Circuit breaker** (FR-42)
  - Do: disable marking, set the breaker threshold to 2, and have the driver comment on its own PR each turn.
  - Pass when: after two consecutive feedback-triggered turns the effort is `paused` and an attention item is raised.
- [ ] **AC-26 Author allowlist** (FR-44)
  - Do: with no allowlist, post a comment as the second account. Then configure an allowlist without that account and post another.
  - Pass when: the first comment starts a turn. The second neither starts a turn nor appears in any later prompt.

### 8.5 Lifecycle safety and recovery

- [ ] **AC-27 Crash recovery** (NFR-5, NFR-8)
  - Do: in separate runs, kill the harness with SIGKILL (a) during spawn and (b) during a turn. Restart it each time.
  - Pass when: after reconciliation, every VM and disk on the host is referenced by the registry. For (a), the effort either finishes spawning or is cleaned up through a defined transition. For (b), the last turn shows "interrupted" and no agent process from that turn remains on the worker (or, if planners chose to re-attach, the turn completes), and a follow-up prompt continues the same session without a second process running alongside it.
- [ ] **AC-28 Age teardown saves work** (FR-18, FR-20)
  - Do: turn the age teardown policy on. Give an effort an open PR, one unpushed commit and one uncommitted change. Let it exceed the age cutoff.
  - Pass when: a salvage branch on the remote contains both pieces of work. The worker and disk are gone, the transcript is on the host, and the state is `torn down`.
- [ ] **AC-29 Revive** (FR-19)
  - Do: after AC-28, post new feedback from the other author, first with the feedback policy on notify and then on auto.
  - Pass when: under notify, an attention item is raised and no worker is created until I use "discuss feedback", which then takes the effort through `reviving` to a new worker. Under auto, the effort goes through `reviving` by itself. In both cases the new worker is on the salvage branch, the session ID is new, and the first prompt includes the PR context.
- [ ] **AC-30 Salvage failure** (FR-20)
  - Do: make the remote unreachable, then let an age teardown run on a dirty worker.
  - Pass when: the worker is kept, an attention item is raised, and teardown is retried once the remote is reachable.
- [ ] **AC-31 Manual teardown** (FR-20)
  - Do: tear down a dirty worker by hand.
  - Pass when: the tool refuses and lists what would be lost. Save-and-tear-down puts the work on the salvage branch. Discard works only after explicit confirmation.
- [ ] **AC-32 Consistent status** (FR-38)
  - Do: run the scenarios above and read the event log. Separately, attach to an effort while it is rate-limited, send it a prompt, then clear the two holds one at a time.
  - Pass when: at every point each effort is in exactly one lifecycle state, its recorded worker condition matches the worker that exists, and no turn starts while a hold applies. With both holds set, both are visible and the prompt starts only after the second is cleared. Every change to the lifecycle state, worker condition or holds appears as an event with a timestamp and cause. After a restart, the registry matches the VMs that exist.

- [ ] **AC-33 Leaving failed** (FR-38, FR-16, FR-20, FR-39)
  - Do: (a) start an effort whose prep step fails. Send it a prompt and start a consult. Fix the prep step with a local override (FR-4) and retry. (b) Using the test hook, make a second effort's spawn fail before its worker boots, then tear it down by hand, and later send it a prompt. (c) Make a third effort fail, then mark it done.
  - Pass when:
    - In (a), the prompt and the consult are refused with the reason, and the effort stays `failed`. After the retry, prep re-runs on the same worker, the effort becomes `active`, and its first prompt runs.
    - In (b), teardown says the check can't run and proceeds only after explicit confirmation. The effort is then `torn down`, and the prompt revives it onto the default branch.
    - In (c), the effort is torn down under the manual rules and becomes `done`.
    - Every transition appears as an event, and none of them is left undefined.

- [ ] **AC-34 Effort with no PR** (FR-39, FR-19, FR-20)
  - Do: turn the age teardown policy on. Start a free-form effort whose turn commits locally and opens no PR. Check the list. Let it exceed the age cutoff. Then send it a prompt. Separately, start a second free-form effort and tear it down by hand while it is clean.
  - Pass when: the list shows "no PR" and a "waiting on me" item after the turn. The harness fetches no feedback for the effort, and starts no turn of its own. After the age cutoff, a salvage branch holds the commit and the state is `torn down`. The prompt moves the effort through `reviving` to a new worker on the salvage branch, with a new session ID and the original prompt in the first prompt. The clean effort revives onto the default branch, since no salvage branch exists. Marking an effort done afterwards leaves it `done`, and sending it a prompt is then refused.
- [ ] **AC-35 PR appears later** (FR-39, FR-40)
  - Do: with the feedback policy set to auto, on an effort with no PR, have a follow-up turn open a PR. Then have the other author comment on it. Separately, on a second effort with no PR, open a PR for its branch from the GitHub web UI, and then link it by hand from the CLI.
  - Pass when: for the first effort, an association event is recorded and the list shows the PR link. The comment starts a turn in the same session. No such comment-driven turn happened before the PR existed. For the second effort, the manual link records an association event and the effort is then treated as a PR effort.

### 8.6 Visibility and the command center

- [ ] **AC-36 List view** (FR-52, FR-53, FR-39, FR-56)
  - Do: with efforts in several states, including one with no PR and one suspended, run the CLI list command and open the command center.
  - Pass when: a single view shows, for every effort, the status (FR-38), PR link (or "no PR"), last turn time, outcome and agent, cumulative cost per agent, pinned master version, disk use, and open attention items. No per-effort command is needed.
- [ ] **AC-37 Live updates** (FR-57)
  - Do: with the command center open, change an effort's status from the CLI.
  - Pass when: the command center shows the change within 5 seconds with no manual refresh.
- [ ] **AC-38 Client independence** (§2.1, FR-63)
  - Do: quit the command center during a running turn. Then do every FR-59 operation from the CLI.
  - Pass when: the turn completes and the registry and events are the same as in a run without the command center. Every operation the command center offers is also available in the CLI.
- [ ] **AC-39 Desktop fit** (§4.10, NFR-7)
  - Do: start the command center on Hyprland on Omarchy and on Fedora's default desktop, with no system tray.
  - Pass when: it runs, and attention notifications arrive on both.

- [ ] **AC-40 Conversation view** (FR-55, FR-29)
  - Do: replay a canned slow turn that emits text and tool calls over about 60 seconds. (a) Open the effort's conversation view from the start. (b) Open it in a second client 30 seconds in. (c) Send a message while the turn is running. (d) Kill the harness with SIGKILL at 40 seconds, restart it and open the view again.
  - Pass when: in (a), items appear within a few seconds of being emitted. In (b), the conversation so far appears, then continues live, with nothing missing or duplicated against the host-side record. In (c), the message shows as queued, then runs after the turn ends in the same session, and the turn record shows it came from the command center. In (d), the output up to the kill is still on the host, no `claude` process from the turn remains on the worker, and the turn is marked "interrupted" (NFR-8), unless planners chose to re-attach, in which case the turn completes. Output the turn produced that had not reached the host at the kill appears in the view after the restart. Viewing changes neither the effort's status nor the turn records.
- [ ] **AC-41 Feedback under notify** (FR-40, FR-41, FR-42)
  - Do: with the default notify policy, have the other author submit a review on PR A and post three loose comments. Then use "discuss feedback" in the command center. Then post one more comment, and have the driver reply with a marked comment.
  - Pass when: no turn starts from the feedback. There is a single attention item for A covering the review and comments, updated rather than duplicated. B and C have none. "Discuss feedback" sends a prompt in A's existing session that includes exactly those items and asks for an assessment without changes. No commit appears. The later comment raises a new item that does not repeat the earlier ones. The driver's marked reply raises nothing.
- [ ] **AC-42 Storage and pinning** (FR-56, FR-11)
  - Do: with workers pinned to two master versions, one of them suspended and one effort torn down, open the storage view in the CLI and the command center. Have one worker write a large file.
  - Pass when: each worker shows its pinned version, whether that is current, and the disk space its clone uses, which matches the host's allocated size for the clone and grows after the write. Each master version shows whether it is current, its dependents and its size. The torn-down effort is listed without disk figures. Pruning a referenced version is refused, and an unreferenced one can be pruned from the view.

### 8.7 Consultant agents

These use Codex as the consultant, on an effort with commits on its branch and an open PR.

- [ ] **AC-43 Review consult** (FR-47, FR-49, FR-51, FR-55, FR-29)
  - Do: after a few exchanges between me and the driver, use the review action with Codex from the command center. Then send the driver a prompt.
  - Pass when:
    - The consult runs on the effort's worker in a new consultant session, separate from the driver's.
    - Its prompt includes my request, the branch, base and PR context, and the environment note. It also includes my messages and the driver's text since the start of the driver's session, without tool calls or their output.
    - The reply appears in the conversation view, labelled as Codex. The turn record names Codex and its cost.
    - No driver turn starts until I send the prompt, and that prompt includes Codex's reply.
    - GitHub shows no push, comment or review from the consult.
- [ ] **AC-44 Consultant session and clearing** (FR-48, FR-49)
  - Do: consult Codex twice, with a driver turn in between. Clear its context, then consult it again.
  - Pass when:
    - The first two consults share one consultant session ID. The second one's prompt carries only the conversation since the first.
    - After the clear, the third consult has a new session ID and the full briefing.
    - The old transcript is on the host, and the conversation view shows the clear. Clearing recorded no turn.
- [ ] **AC-45 Read-only enforced** (FR-50, FR-53)
  - Do: (a) using the test hook, run `gh auth status` and `git push` in a consult's environment. (b) Using the test hook that lets a consult write, have a consult edit a tracked file and commit. Then revert the change from the CLI and resume.
  - Pass when:
    - In (a), both fail and the working tree is unchanged.
    - In (b), an event is recorded, the effort is `paused`, and an attention item shows the changed file and the new commit. No driver turn starts while it is paused.
    - After the revert, HEAD and the file are as before the consult.
- [ ] **AC-46 Queue and holds** (FR-30, FR-33, FR-34, FR-38, FR-47)
  - Do: (a) start a consult while a driver turn is running. (b) Stub a usage-limit failure for Codex, then send the driver a prompt and start another consult. (c) Attach (FR-31), then start a consult. (d) Abort a running consult, resume the effort, and consult again.
  - Pass when:
    - In (a), the consult is queued and runs after the driver turn. No two turns overlap.
    - In (b), Codex shows `rate-limited` on the effort, the driver turn still runs, and the consult runs after the stubbed reset.
    - In (c), the consult is held until I detach.
    - In (d), the outcome is "aborted" and the effort is `paused`. Starting a consult does not resume it. After I resume, the next consult continues the same consultant session.
- [ ] **AC-47 Availability and refusal** (FR-24, FR-46, FR-47)
  - Do: refresh the master with a consultant set to fail the smoke test (for example with an invalid credential). Spawn a worker on the new version and consult that consultant. Separately, start a consult on a `torn down` effort and on a `done` effort.
  - Pass when:
    - The refresh succeeds and reports the consultant as left out, with the reason.
    - The consult on the new worker is refused with the reason.
    - Both consults on efforts without a worker are refused, and neither effort changes state.
- [ ] **AC-48 Teardown and revival** (FR-19, FR-48)
  - Do: consult Codex on an effort, tear the effort down, revive it by sending the driver a prompt, then consult Codex again.
  - Pass when:
    - After teardown, both the driver's and Codex's transcripts are on the host.
    - After the revival, the consult starts a new consultant session with the full briefing.

### 8.8 Hosts and measurements

- [ ] **AC-49 Both distros** (NFR-7)
  - Do: install on a clean Omarchy host and a clean Fedora host using the documented steps (FR-2), then run all the criteria above.
  - Pass when: every criterion passes on both.
- [ ] **AC-50 Measurements** (NFR-2, NFR-3)
  - Do: spawn 10 workers one after another. Record the time to SSH-ready and each fresh clone's allocated size against its master.
  - Pass when: the median and worst times and the sizes are recorded. This is informational. It sets the target in §7.
