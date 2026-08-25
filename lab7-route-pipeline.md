# Cribl Stream: Creating a Route and a Pipeline (UI Walkthrough)

This guide walks through the two screens used to create a new **Data Route** and a new **Pipeline** in the Cribl Stream UI (Worker Groups > `default`).

---

## 1. Creating a Data Route

**Navigation:** `Worker Groups > default > Routing > Data Routes`

The Routing tab sits alongside **Overview**, **Data**, **Processing**, **Projects**, and **Worker Group Settings**.

### 1.1 The Routes table

- Click **Add Route** (top right) to create a new row in the routing table.
- Each Route appears as a numbered row (e.g. **2**) with:
  - A drag handle (⠿) to reorder — remember, **Routes are evaluated top to bottom**, so position determines precedence.
  - An **enable/disable toggle** (blue = enabled).
  - A warning icon (⚠) if the Route is incomplete/misconfigured — visible here because required fields haven't been filled in yet.
  - A live **Events (In)** percentage column showing what share of traffic is hitting this Route.
  - A **⋯** menu for row-level actions (clone, delete, etc.).

### 1.2 Route configuration fields

When you expand a Route row, you get:

| Field | Purpose |
|---|---|
| **Route name*** | Required. A unique, descriptive name for this Route. |
| **Filter** | A JS filter expression (defaults to `true`, matching all events). Click the expand icon (⤢) to open it in a larger editor, or the chevron (⌄) for more options. |
| **Pipeline*** | Required. Select which Pipeline processes events that match this Route's filter. |
| **Enable expression** | Toggle on to use a dynamic expression (instead of a static Destination) to compute the output at runtime. |
| **Destination** | Select the Destination events should be sent to after Pipeline processing. |
| **Description** | Optional free-text notes on the Route's purpose. |

### 1.3 Saving

- **Cancel** discards changes.
- **Save** commits the Route to the routing table (still requires **Commit & Deploy** afterward to push live to Worker Groups).

> Tip: Leaving Filter as `true` with no other Routes above it makes this a catch-all/default Route — useful as a final fallback, but be careful about where you place it if you have more specific Routes to evaluate first.

---

## 2. Creating a Pipeline

**Navigation:** `Worker Groups > default > Processing > Pipelines`

### 2.1 Pipeline metadata fields

| Field | Purpose |
|---|---|
| **ID*** | Required. The Pipeline's unique name — this is what you'll select from the **Pipeline** dropdown when configuring a Route. |
| **Async function timeout (ms)** | Default `1000`. Maximum time an async Function in this Pipeline is allowed to run before timing out. |
| **Description** | Optional notes on what this Pipeline does. |
| **Tags** | Optional tags for organizing/filtering Pipelines in the UI. |

### 2.2 Building and testing the Pipeline

Once the ID is set, you build out the Pipeline's Functions in the main canvas (not shown in the metadata screen). Along the top of the editor:

- **▶ (Run)** — executes the Pipeline against sample or live data so you can preview IN vs. OUT results as you add Functions.
- **Sample Data panel** (right side) — lets you:
  - **Import Data** — upload a sample file to test against.
  - **Search samples** — find previously saved sample sets.
  - Browse the **Samples** tab, which lists sample files by name for repeated testing.

This is the same **Data Preview** mechanism used to validate Pipeline logic before attaching it to a live Route — use it to confirm the Pipeline produces the expected output before saving.

### 2.3 Saving

- **Cancel** discards the new Pipeline.
- **Save** is greyed out until required fields (ID) are filled in — once enabled, it creates the Pipeline so it becomes selectable in a Route's **Pipeline** dropdown.

---

## 3. How These Two Screens Connect

```
Create Pipeline (Processing tab)
        │
        ▼
Pipeline now appears in the "Pipeline*" dropdown
        │
        ▼
Create/Edit Route (Routing tab) → set Filter → select Pipeline → select Destination → Save
        │
        ▼
Commit & Deploy to push the Route + Pipeline live to Worker Groups
```

In practice, it's common to build and test the Pipeline first (using Sample Data/Run), then create the Route that references it — that way the Pipeline dropdown isn't empty and you can validate output before wiring it into live routing.

---

*Based on the Cribl Stream Data Routes and Pipelines configuration screens (Worker Groups > default). UI labels may vary slightly by version.*
