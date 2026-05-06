# GDOT Plan Tools

**Browser-based productivity tools built for civil engineers working on Georgia DOT projects.**  
No install. No login. Open the file, get to work.

---

## What's Inside

| Tool | Purpose |
|---|---|
| **FlowBoard** | Kanban project board with real-time cloud sync |
| **Section 24 Checklist** | Utility Plans — PPG compliance checklist |
| **Section 50 Checklist** | Erosion, Sedimentation & Pollution Control — PPG compliance checklist |
| **Section 60 Checklist** | Right of Way Plans — PPG compliance checklist |

---

## FlowBoard

A fast, focused Kanban board designed for managing the moving parts of GDOT project deliverables — submittals, reviews, revisions, coordination tasks, and everything in between.

**Core features**

- **Groups & columns** — Organize by project, phase, or discipline. Columns represent workflow stages (To Do → In Progress → Review → Done)
- **Cards with depth** — Each task carries a title, description, color label, priority flag, tags, due date, and a sub-item checklist
- **WIP limits** — Cap work-in-progress per column to surface bottlenecks before they become delays
- **Drag & drop** — Reorder cards and columns with fluid animations; collapsed columns pin as vertical pills to save screen space
- **Deadline tracking** — Overdue cards pulse red so nothing slips past a submission date
- **Search** — Filter cards across all groups instantly
- **Undo / Redo** — Full history stack; mistakes are reversible
- **15 themes** — Dark and light palettes including Midnight, Navy, Pearl, and more
- **Cloud sync** — Firebase Realtime Database keeps your board identical across devices; a safety gate blocks accidental overwrites from stale data
- **Backups** — Manual snapshots, auto-snapshots on idle, export/import JSON; up to 50 snapshots stored
- **Archive** — Move completed columns out of view without deleting them

**Works entirely in the browser** — save the HTML file locally or host it on any static server.

---

## PPG Compliance Checklists

The GDOT [Plan Presentation Guide](https://gdot.ga.gov) defines exactly what must appear on every plan sheet before a set is accepted. Missing an item means a rejection, a resubmittal, and lost time.

These checklists put every PPG requirement in front of you as an interactive list — organized by section, with the original wording preserved verbatim. Check items off as you review the plans; the progress bar tracks where you stand.

**Section 24 — Utility Plans**  
**Section 50 — Erosion, Sedimentation and Pollution Control Cover Drawing**  
**Section 60 — Right of Way Plans** (Cover Drawing + Plan Drawings General)

**Checklist features**

- **Filter view** — Show All, Remaining only, or Done — focus on what's left without losing context
- **Progress tracking** — Live counter and progress bar per section and overall; sections badge green when complete
- **Collapsible sections** — Collapse sections you've finished to reduce visual noise
- **Auto-save** — Every check is written to localStorage instantly; reopen the file and pick up exactly where you left off
- **Snapshots** — Save named backups of your progress at any point; restore with one click
- **Export / Import** — Transfer progress between machines via JSON; useful when plans move between workstations or team members
- **Save as PNG / PDF** — Export a clean image or print-ready PDF of your completed checklist for the project file

---

## How They Work Together

1. **Open FlowBoard** and create a column for each phase of your submittal — *In Progress*, *Internal QC*, *Checking PPG*, *Ready to Submit*
2. When a plan set reaches *Checking PPG*, **open the relevant checklist** and work through it item by item
3. Once the checklist is complete, drag the card to *Ready to Submit* and export a PDF of the checklist as documentation
4. Reopen both files for the next review cycle — progress is remembered

---

## Quick Start

All tools are single HTML files. No build step, no dependencies to install.

1. Clone or download this repository
2. Open any `.html` file directly in Chrome, Edge, or Firefox
3. For FlowBoard cloud sync, add your own Firebase Realtime Database credentials in the config block at the top of `flowboard.html`

---

## Who This Is For

Civil engineers, designers, and project managers at GDOT and consulting firms preparing construction plan sets for Georgia DOT submission. The checklists are keyed directly to the PPG — if you work on GDOT projects, these sections apply to your plans.
