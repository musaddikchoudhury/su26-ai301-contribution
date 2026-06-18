# su26-ai301-contribution
# Contribution 1: feature request: options to display only positive or negative x and y axes

**Contribution Number:** 1  
**Student:** Musaddik Choudhury  
**Issue:** [https://github.com/Doenet/DoenetML/issues/355](https://github.com/Doenet/DoenetML/issues/355)  
**Status:** Phase II Complete

---

## Lessons Learned from Previous Issues

My original issue was [SwitchbackTech/compass #1759 — "Someday event title does not update while form is open"](https://github.com/SwitchbackTech/compass/issues/1759). Before I could begin Phase II, it was closed and fixed by another contributor.

I then selected [betterlaspinas/betterlaspinas #60 — "UI Overlap and Responsive Layout Regression on Mobile Viewports"](https://github.com/betterlaspinas/betterlaspinas/issues/60). After claiming it, I learned someone else had already been working on it, so I needed to select another issue.

**What I learned:** In open source, issues can move fast and the "Claimed?" status isn't always obvious from comments alone. Before committing time to an issue, I now check more carefully for any indication of existing work, and I try to get direct confirmation from a maintainer that an issue is genuinely available before investing further. Per my instructor's guidance, I have updated this README with my new issue while keeping this as Contribution 1.

---

## Why I Chose This Issue

I chose issue #355 "feature request: options to display only positive or negative x and y axes" because it is a well-scoped feature with a maintainer-provided implementation roadmap, which gave me confidence I could realistically contribute to it. I commented on the issue, and `dqnykamp`, a core maintainer, responded with specific guidance — naming the exact files involved (`Graph.js` in the core package and `graph.tsx` in the renderer), pointing me to an existing similar attribute pattern (`displayMode` and the `simplify` attribute in `Math.js`) to model my implementation after, and offering to help if I get stuck.

This matches my skills and goals well. I have experience with React-based component architecture from projects like FocusMind and BridgeAI, so I'm comfortable navigating component logic and state. DoenetML is an educational math platform built by a university team, and I'm interested in learning how a project like this separates its core logic from its rendering layer, since that's a pattern I haven't worked with directly before. The clear maintainer engagement and concrete starting point made this feel like the strongest opportunity among the issues I considered.

## Understanding the Issue

### Problem Description

The `<graph>` component's `displayXAxis` and `displayYAxis` attributes currently only support `true`/`full` and `false`/`none`. There is no way to display only the positive or only the negative portion of an axis, which limits how the graph can be used in math contexts like absolute value or distance problems.

### Expected Behavior

Setting `displayXAxis="positiveOnly"` should render only the positive half of the x-axis (line, arrow, ticks, and labels for values greater than zero). `displayXAxis="negativeOnly"` should do the equivalent for the negative half. The same should apply to `displayYAxis`. The axis should update dynamically if the attribute value changes.

### Current Behavior

Setting `displayXAxis="positiveOnly"` does not throw an error, but the x-axis disappears entirely — no axis line, arrow, tick marks, or numeric labels render at all. I confirmed this by comparing two graphs side by side: one with `displayXAxis="true"` (full axis rendered correctly) and one with `displayXAxis="positiveOnly"` (axis missing completely). This means the attribute is being treated as falsy for any value other than the literal string `"true"`, rather than being recognized as a distinct intermediate state.

### Affected Components

DoenetML has three layers, and this bug touches two of them:

- **Worker logic (JavaScript):** `packages/doenetml-worker-javascript/src/components/Graph.js` — this is where the `displayXAxis`/`displayYAxis` attribute is defined and where its value gets processed before being sent to the renderer
- **UI renderer (React):** `packages/doenetml/src/Viewer/renderers/graph.tsx` — this is where the actual axis gets drawn on screen based on the value it receives
- **Reference pattern:** `packages/doenetml-worker-javascript/.../Math.js` — the `simplify` attribute here already solves a similar problem (a multi-value attribute instead of a simple true/false), so I'll copy its pattern

Note: there's also a Rust/WASM layer in the middle of this system that handles some reference resolution, but the maintainer's guidance points specifically at the JavaScript files above, so I don't expect to need to touch Rust code for this fix.

---

## Reproduction Process

### Environment Setup

I used GitHub Codespaces to avoid known Windows compatibility issues with the local Rust/WASM build (documented in the project's README and PR #326). I created a Codespace on my fork, which automatically ran the `postCreateCommand` setup script to install Node, Rust, and all workspace dependencies. Once setup finished, I ran `npm run dev` from the root directory, which started a development server. I accessed it through the forwarded port link, which opened a test viewer with a code editor on the left and a live preview on the right.

**Important notes I found in the project's `AGENTS.md` contributor guide that I'm following:**
- The test viewer file (`packages/doenetml/dev/testCode.doenet`) is meant to be edited locally for experimentation, but it should never be staged or committed — it's a personal scratchpad, not part of the actual project
- Code must be formatted with `npm run prettier:format` before any commit
- Tests must be run with `--run` (one-shot) rather than watch mode, and Cypress requires a rebuild before every run to avoid testing stale code

Working branch: https://github.com/musaddikchoudhury/DoenetML/tree/fix-issue-355

### Steps to Reproduce

1. Open the DoenetML test viewer in a running dev environment (`npm run dev`, then open the forwarded port)
2. In the code editor, enter the following DoenetML:
   ```xml
   <graph displayXAxis="true" displayYAxis="true">
       <point xs="2 3"/>
   </graph>

   <graph displayXAxis="positiveOnly" displayYAxis="true">
       <point xs="2 3"/>
   </graph>
   ```
3. Click "Update" to render the preview
4. **Expected:** The second graph shows only the positive half of the x-axis
5. **Actual:** The second graph's x-axis disappears entirely — no axis line, arrow, ticks, or labels are rendered

### Reproduction Evidence

- **Branch:** https://github.com/musaddikchoudhury/DoenetML/tree/fix-issue-355
- **Screenshots:** Confirmed via side-by-side comparison in the test viewer — first graph (`displayXAxis="true"`) renders a complete axis from -8 to 8; second graph (`displayXAxis="positiveOnly"`) renders no x-axis at all
- **My findings:** The attribute parser is not recognizing `positiveOnly` as a valid value and is defaulting to falsy/hidden behavior instead of either showing the full axis or throwing a validation error. This confirms the root cause is in how `displayXAxis`/`displayYAxis` values are interpreted, not in the rendering logic itself.

---

## Solution Approach

### Analysis

The root cause is in how `Graph.js` processes the `displayXAxis`/`displayYAxis` attribute value. Currently the attribute logic only maps two states (true/false), so any other string value falls through to the falsy case and the axis is hidden. To support `positiveOnly` and `negativeOnly`, the attribute needs to recognize four distinct states instead of two, and the renderer needs to be told which state is active so it can draw the axis accordingly (full line, half line, or none).

### Proposed Solution

Extend the `displayXAxis`/`displayYAxis` attribute definitions in `Graph.js` to recognize four values (`full`/`true`, `none`/`false`, `positiveOnly`, `negativeOnly`) instead of two, following the same pattern used by the `simplify` attribute in `Math.js`. Pass the resolved state through to the renderer (`graph.tsx`) via the existing `forRenderer: true` mechanism, and update the renderer's axis-drawing logic to handle the two new states by configuring the underlying graphing library's axis options (e.g., `straightFirst`/`straightLast` or equivalent) so only the relevant half of the axis renders.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The `displayXAxis`/`displayYAxis` attributes only support full/none states. They need to also support showing only the positive or only the negative half of each axis, updating dynamically if the attribute changes.

**Match:** The `simplify` attribute in `Math.js` already handles a similar multi-value boolean-like attribute using `valueForTrue`/`valueForFalse` fields. The existing `attributes.displayMode` definition in `Graph.js` shows the current pattern for `displayXAxis`/`displayYAxis` that needs to be extended.

**Plan:**
1. In `Graph.js`, find the `displayXAxis`/`displayYAxis` attribute definitions and extend them to recognize `"positiveOnly"` and `"negativeOnly"` as additional valid values, copying the pattern already used by the `simplify` attribute in `Math.js`
2. Confirm the resolved value correctly reaches the renderer (it should already be marked `forRenderer: true` — I'll verify this still works for the new values)
3. In `graph.tsx`, update the axis-drawing logic to read the new states and configure the graphing library so it draws only the positive or negative half of the axis when needed
4. Confirm the axis updates immediately if the attribute value changes while the page is open, not just on first load
5. Write tests covering all four states (`full`, `none`, `positiveOnly`, `negativeOnly`) for both `displayXAxis` and `displayYAxis`
6. Format all changed files with `npm run prettier:format` before committing
7. Follow the function-declaration and async/await style conventions described in the project's `AGENTS.md` guide

**Implement:** Branch created at https://github.com/musaddikchoudhury/DoenetML/tree/fix-issue-355 — implementation begins in Phase III

**Review:** I will run `npm run prettier:format`, double check I haven't accidentally staged my personal test file (`packages/doenetml/dev/testCode.doenet`), and confirm my PR is based on `upstream/main` as the project's contributor guide requires

**Evaluate:** I will manually test all four attribute combinations in the test viewer, confirm the axis updates dynamically, run the targeted Vitest suite for the affected package with `--run`, and rebuild before any Cypress test to avoid stale results

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
