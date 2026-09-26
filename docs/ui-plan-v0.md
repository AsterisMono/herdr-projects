# Web UI v0 plan

A plan for a local web dashboard that watches Herdr agents, project by project. Nothing here is built. This file covers the product and the screens only; the implementation (TypeScript: a Node coordinator with SQLite and a Vite web UI) gets its own plan later.

Prototype: [`prototypes/ui-v0.html`](prototypes/ui-v0.html). Open it in a browser from a checkout. It needs no build and no network, and every piece of data in it is made up.

## What v0 is for

You run several coding agents in Herdr and want one browser tab that tells you which projects exist, which agents each one runs, what state each agent is in, and what the one you pick is printing. v0 answers "what is happening?" and nothing else. Every change to an agent (starting it, prompting it, answering its question) still happens on the command line or in the Herdr TUI.

## Where it sits

- **Separate from the plugin.** The plugin in this repository keeps its record in project folders and has no database. The dashboard is its own local app with its own SQLite file, and it does not read or write `~/.herdr-projects/`.
- **Herdr owns the sessions.** The dashboard never runs a terminal. Status and terminal output come from Herdr each time they are shown.
- **Local only.** One person, on the machine that runs Herdr. No accounts and no sharing.

## Locked decisions

Settled before this plan; the plan builds on them and does not revisit them.

| Topic | Decision |
| --- | --- |
| Runtime | Herdr owns every session, its status and its I/O. No custom PTY, no herdr-web. |
| Record | SQLite with two tables, `projects` and `runs`. No `events`, no `shared_context`. |
| Status | Only from Herdr: `idle`, `working`, `blocked`, `done`, `unknown`. SQLite never stores or caches it. |
| Terminal | Observe-only: `herdr terminal session observe` → WebSocket → xterm.js. No input, resize, scroll or takeover; `terminal session control` is not used. |
| Steering | Not from the web: no prompt box, no follow-up, no keys. Command line and Herdr TUI only. |
| Herdr layout | 1 project ↔ 1 workspace. 1 Agent ↔ 1 pane ↔ 1 tab; never two agents in one tab. |
| Naming | **Coordinator** and **Agents**, as in Cursor Projects. |
| Screen layout | Left: Projects with their Agents nested under them. Centre: the Coordinator, a stub in v0. Right: the selected Agent's status and terminal. |
| Out of scope | Personal-site pulse, web steer, Listening, Automations, pull request chrome, Take control. |

The record, exactly:

```
projects (id, name, path, herdr_workspace_id, created_at)
runs     (id, project_id, herdr_pane_id, started_at)        -- no status column
```

- `herdr_workspace_id` is the `workspace_id` Herdr returns from `workspace create`, such as `w1`.
- `herdr_pane_id` is the id of the root pane of the Agent's own tab, as Herdr returned it, such as `w1:p7`. Pane numbers need not match tab numbers, so the UI never builds one id from another.
- There is no tab id, agent name, title, kind or status column. Anything the UI shows beyond these columns is read live from Herdr.

## Naming

| In the UI | What it is | Record | In Herdr |
| --- | --- | --- | --- |
| **Project** | A body of work in one folder | a `projects` row | one workspace (`herdr_workspace_id`) |
| **Coordinator** | The project's control plane and, later, the one you chat with. It plans and starts Agents; it does not do the work itself. | none in v0 | not shown in v0 |
| **Agent** | One worker on one task | a `runs` row | one pane (`herdr_pane_id`), alone in its own tab, with an agent started in it |

Rules for the UI's words:

- Say "Agent". "Run" is the table's name and stays in code. Never call an Agent a project, a session or a thread.
- The plugin calls its workers *threads*. The dashboard does not use that word, so the two are not confused.
- "Coordinator" names the project-level surface only, never a single Agent.

## Information architecture

```
Projects                     left column, in the order they were created
└── Project                  name, Agent count
    └── Agents               newest first, status or time on each row
        └── Agent detail     right column: status header, read-only terminal
Coordinator                  centre column: the selected project's Coordinator (stub)
```

One page with three columns and no other pages. The address holds the selection, so a reload or a bookmark comes back to it:

| Address | Shows |
| --- | --- |
| `#/` | The first project, no Agent selected |
| `#/projects/<id>` | That project, no Agent selected |
| `#/projects/<id>/agents/<run id>` | That project and that Agent |

An id that does not exist falls back to the level above it.

## Layout

```
┌──────────────────────────────┬────────────────────────────┬────────────────────────────────────────┐
│ Projects                     │ ▣ Website v2  Coordinator  │ Fix bug report                 Working │
├──────────────────────────────┼────────────────────────────┼────────────────────────────────────────┤
│                              │                            │ fix-bug-report · claude · w1:p9 [copy] │
│ ▣ Website v2               8 │             ▣              │ ┌────────────────────────────────────┐ │
│   ◐ Fix bug report   Working │         Website v2         │ │ read-only terminal (xterm.js)      │ │
│   ◐ Finalize testi…  Working │       ~/dev/website        │ │                                    │ │
│   ● Migrate forms t… Blocked │  8 agents · 2 working · …  │ │                                    │ │
│   ● Tune Lighthouse…      2h │                            │ │                                    │ │
│   ● Update footer l…      3h │ No Coordinator chat in v0. │ │                                    │ │
│   Show 3 more                │ Start and steer agents     │ │                                    │ │
│ ▣ Gardener                 3 │ from the command line or   │ │                                    │ │
│ ▣ Browser                ● 2 │ the Herdr TUI.             │ │                                    │ │
│ ▣ Design System            0 │                            │ │                                    │ │
│                              │ Details                    │ ├────────────────────────────────────┤ │
│ ● Herdr connected            │ Path       ~/dev/website   │ │ Live · Read-only · 104×38          │ │
│                              │ Workspace  w1              │ └────────────────────────────────────┘ │
│                              │                            │                                        │
│                              │ Observe-only · steer…      │                                        │
└──────────────────────────────┴────────────────────────────┴────────────────────────────────────────┘
```

- Widths: the left column is fixed at about 264 px and the centre at about 380 px. The right column takes the rest, because a terminal needs at least 80 columns.
- Each column is its own glass pane, and their headers share one height, so the three read as one window.
- Desktop first. Below about 1,100 px the centre column is hidden. There is no phone layout in v0.
- One light theme, described next.

### Visual direction

The skin blends two references and borrows one small thing from a third. It changes only the look; the layout, states and behaviour in this plan stay as they are.

- **[Frutiger Aero](https://en.wikipedia.org/wiki/Frutiger_Aero) materials set the mood.** A calm sky-to-aqua wallpaper with a faint aurora, a leaf-green horizon, a few soap bubbles and light noise. Glass, gloss and soft gradients in cyan, teal, sky and leaf green.
- **[Windows 7 Aero](https://en.wikipedia.org/wiki/Windows_Aero) chrome is the specimen.** Each column is a frosted translucent pane (backdrop blur over translucent white) with a 1 px bright rim, a soft inner highlight and a soft shadow, rounded at 14 px. Wallpaper shows in the gaps between panes instead of grey dividers. Column headers are glossy strips, and a seam inside a pane is a teal hairline over a white highlight. One humanist sans throughout: Segoe UI, then Inter or the system font.
- **One [88×31](https://indieweb.org/88x31) button**, "Herdr · observe only", sits in the Coordinator's footer. It is the only piece of the personal-web layer.

| Element | Treatment |
| --- | --- |
| Status pill | A gel capsule, a glossy top half over a tinted body: working aqua and teal, blocked warm amber, done soft green, idle and the other quiet states frosted grey. The word is always in the capsule. |
| Row icons | Glossy orbs in the same colours, a teal spinner, a hollow ring, a dash |
| Live | A small glossy green gel in the terminal footer |
| Blocked | An amber glass line under the detail header |
| Unreachable | A muted orange glass banner across the centre and right panes |
| Selection | Brighter glass with a cyan rim, strongest on the selected Agent and softer on its project. Never a flat blue fill. |
| Coordinator note | A softer frosted card |
| Terminal | A dark glass inset with light text. Monospace is used here and nowhere else, and logs never sit on the wallpaper. |

- Text meets WCAG AA contrast on the glass: the panes are opaque enough that the wallpaper never decides whether text is readable. The one exception is the dimmed last frame of a terminal that is reconnecting or closed.
- Motion is limited to 150–300 ms ease-out fades and highlights, with no bounce. Under reduced motion nothing animates, the spinner included.
- Not used: a Start orb, caption buttons, Flip 3D, GeoCities density, glitter tiles, marquees, background music, Nightcore or anime imagery, Y2K liquid metal.

Tokens:

```css
--sky: #cfefff;   --aqua: #7ec8e3;   --teal: #3aa6a0;   --leaf: #7cbc6e;
--glass: rgba(255, 255, 255, .55);   --ink: #1a2a33;   --blocked: #e8a05c;   --live: #3cba6f;
```

## Screens

### Left: Projects and Agents

Project row:

- A small square with the project's first letter, its colour picked from the project id (there is no icon column).
- The name.
- An amber dot when any of its Agents is `blocked`, so a project that needs you stands out even when folded.
- A count: the number of its Agents, meaning all its `runs` rows, including those whose pane is gone.

Clicking a project selects it. The centre shows its Coordinator, its Agents unfold under it, and the Agents of any other project fold away. Clicking the selected project again folds or unfolds its Agents. Selecting another project clears the Agent selection. There is no "+" button, because v0 creates nothing from the web.

Agent row, nested under its project:

- Status icon, title, and on the right a status word or a time (see [Status](#status)).
- Newest first by `started_at`. A status change never reorders the rows.
- The first five show; "Show N more" reveals the rest. A selected Agent is always visible.
- Clicking selects it and opens it on the right.

An Agent's title, first match wins, all read live from Herdr:

1. The label of the Agent's tab, if the Coordinator set one when it created the tab.
2. The Herdr agent name, written as words: `fix-bug-report` becomes "Fix bug report".
3. `Agent #<run id>`, when Herdr no longer has the pane.

The raw agent name is on the detail header, since that is what you type on the command line.

At the bottom of the column: the Herdr connection, "Herdr connected" or "Herdr unreachable".

Keyboard: every row is a button in tab order, ↑ and ↓ move between rows, Enter selects.

### Centre: Coordinator (stub)

v0 has no Coordinator chat, but the column is there now so the layout does not change when chat arrives. It shows:

- The project's square, name and path, with the home folder written as `~`.
- A live summary of its Agents: how many there are, then how many are working, blocked, done and idle, leaving out zeros: `8 agents · 2 working · 1 blocked · 1 done · 1 idle`.
- A short note: "No Coordinator chat in v0. Start and steer agents from the command line or the Herdr TUI; this page shows what they are doing."
- Details: the path, the Herdr workspace id, the created date, the number of Agents and when the newest started.
- Where a message box would be, one quiet line: "Observe-only · steer from Herdr". No text box, not even a disabled one.

### Right: Agent detail

Header:

- The Agent's title and its status pill.
- One line of facts: the Herdr agent name, the agent kind when Herdr reports one (such as `claude`), the pane id with a copy button, and "started 18 minutes ago".
- A line under it for three states: `blocked` ("Waiting on you. Answer it in Herdr; the web is read-only."), `unknown` ("Herdr can't tell what this agent is doing.") and exited ("The agent has exited. Its pane is still open.").

Terminal:

- A read-only xterm.js showing the pane through `herdr terminal session observe`. Typing and pasting do nothing except show "Read-only. Type into this agent from Herdr." Selecting and copying text works.
- Scrolling moves through what the browser has received since it opened the Agent. It never scrolls the pane.
- The terminal fills the column and passes its size to observe as columns and rows. It never resizes the pane.
- A footer with the stream's state, "Read-only" and the size: `Live · Read-only · 104×38`.

There are no tabs (no Git, Desktop or Files) and no button that acts on the Agent.

Terminal states:

| State | When | The terminal area shows |
| --- | --- | --- |
| Connecting | From opening an Agent until the first frame | "Connecting to w1:p9…" |
| Live | Frames arriving | The terminal |
| Reconnecting | Herdr is unreachable, or the stream stopped without `terminal.closed` | The last frame, dimmed, and "Reconnecting…" |
| Closed | `terminal.closed` arrived | The last frame, dimmed, and "The pane closed." |
| Nothing to show | Herdr has no pane with this id | "This pane is no longer in Herdr." |

### Empty states

| Where | When | Text |
| --- | --- | --- |
| Whole page | No projects | "No projects yet. They appear here when the Coordinator's command line creates them." |
| Under a project | It has no Agents | "No agents yet" |
| Right | A project is selected but no Agent | "Select an agent to watch its terminal." |
| Right | The project has no Agents | "Agents appear here when the Coordinator starts them." |

## Status

Herdr is the only source. Rows, the project summary and the detail pill show what Herdr reports when they are drawn and change when Herdr reports a change. Nothing is written to SQLite. Three states belong to the UI, not to Herdr: they describe Herdr's answer, not the agent.

| State | From | Row icon | Row, right side | Detail pill |
| --- | --- | --- | --- | --- |
| `working` | Herdr | teal spinner | Working | Working |
| `blocked` | Herdr | amber dot | Blocked | Blocked |
| `done` | Herdr | green dot | time, such as `2h` | Done |
| `idle` | Herdr | grey dot | time | Idle |
| `unknown` | Herdr | hollow circle | Unknown | Unknown |
| Exited | UI: the pane is there, but no agent runs in it | hollow circle | Exited | Exited |
| Not found | UI: Herdr has no pane with this id | dash | Not found | Not in Herdr |
| Unreachable | UI: the dashboard cannot reach Herdr | none | — | Herdr unreachable |

- **The time** on `done` and `idle` rows is the time since `runs.started_at`, the only time the record holds: `now`, `12m`, `2h`, `3d`. The tooltip and the detail header spell it out as "started 2 hours ago".
- **Blocked** is the only state that asks something of you. It gets the amber dot on the project row, the word on the Agent row, and the line on the detail header that says where to answer.
- **Unreachable** shows one banner across the centre and right columns: "Can't reach Herdr. Statuses and terminals are paused until it's back." Rows keep the last title they showed and read `—`.
- **Colour is never the only signal.** Each icon has its word beside it or in its tooltip and accessible name, and the pill always has the word.
- The spinner stops when the system asks for reduced motion.

## What the screens read

This is not a backend design, only what each element reads, so the implementation plan can check that it provides it.

| Element | Comes from |
| --- | --- |
| Project name, path, created date, workspace id | `projects` |
| Agent count, order, "started …" | `runs` (`project_id`, `started_at`) |
| Agent title, agent name, agent kind | Herdr, live: the pane's tab label, the agent's name and kind |
| Status on rows, summary and pill | Herdr, live: the agent list and status-change events |
| Terminal | `herdr terminal session observe <herdr_pane_id> --cols N --rows N`, streamed over a WebSocket |

The web UI writes nothing: no request creates, changes or deletes anything, in SQLite or in Herdr. Field names in Herdr's replies beyond the ones this repository has verified (`workspace_id`, `pane_id`, `agent_status`) should be checked with `herdr api schema --json` when this is built.

## Out of scope for v0

Not in the plan and not in the prototype:

- Coordinator chat. The centre column is a stub.
- Any web steer: prompts, follow-ups, a message box, sending keys, pasting into a terminal, answering a blocked Agent.
- Terminal control: `herdr terminal session control`, resizing or scrolling the pane, Take control.
- Creating, renaming, stopping or deleting projects or Agents from the web. This plan decides it; the create lifecycle belongs to the Coordinator's command line.
- The Listening pill, subscriptions and Automations.
- A track board (Ready, In progress, …).
- Pull request, diff, review and merge chrome, and Git, Desktop or Files tabs.
- Agent transcripts, artifacts and screenshots.
- The personal-site pulse.
- Accounts, sharing and remote access; a dark theme; a phone layout; notifications.
- herdr-web, Ghostty Web or libghostty as the terminal.

## The prototype

[`prototypes/ui-v0.html`](prototypes/ui-v0.html) is one self-contained file: no build, no network, no dependencies.

It shows:

- Four projects and thirteen Agents as mock `projects` and `runs` rows shaped like the schema, next to a separate mock of what Herdr would report for each pane.
- Every state in the status table, including an exited Agent, a pane Herdr no longer has, and `unknown`.
- A project with no Agents, a project with more than five, and the fallbacks for an Agent's title.
- Addresses such as `#/projects/1/agents/8`, so a reload keeps the selection.
- ↑ and ↓ in the Projects column, the copy button, and the read-only message when you type into a terminal.

What is fake:

- The terminal is a styled text block fed by canned output, not xterm.js. Working Agents print a new line every few seconds.
- The "Prototype" button at the bottom of the Projects column opens controls that are not part of the product: move the selected Agent to its next status, close its pane, make Herdr unreachable, show an empty record, and reset.

## Open questions

1. **Pane ids come back after a Herdr server restart. Deferred; a known risk in v0.** Herdr numbers workspaces and panes from `w1` again after a restart ([`herdr-notes.md`](herdr-notes.md)), so a stored `herdr_pane_id` can name a different pane. An Agent's row can then show another pane's status and terminal. The plugin guards against this with an identity check (ids, working directory and agent name); v0 does not. Rematching is deferred entirely: v0 adds no rule and no column for it, and the schema stays as locked.
2. **The time on done and idle rows.** v0 uses `started_at`. If Herdr's status-change events carry a time, the time since the last change would match what people expect better. It would be held in memory only, never in SQLite.
3. **Titles.** A readable title depends on the Coordinator labelling each Agent's tab, or naming its agent, when it creates it. Once the tab is closed only `Agent #<run id>` is left.
4. **Creating from the web.** Out of scope now. If it comes, it follows the same lifecycle as the command line: insert the row, create in Herdr, write the id back, delete the row if Herdr fails.
5. **The plugin's projects.** Whether the dashboard should ever show the plugin's projects and threads is open. Their layout differs: a worktree thread has its own workspace, so a plugin project is not one workspace.

## Where the patterns come from

The layout follows [Cursor Projects](https://cursor.com/docs/agent/projects): a Projects list in the left column with the Agents under each project, the Coordinator for the selected project, and a detail column for one Agent, like the terminal column of Cursor's web agent dashboard. Taken: the hierarchy, the names, the row shape (a status word while working, a time otherwise) and status on the detail. Left out: everything under [Out of scope](#out-of-scope-for-v0). The look does not come from Cursor; see [Visual direction](#visual-direction).
