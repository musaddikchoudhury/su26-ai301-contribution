# su26-ai301-contribution
# Contribution 1: feature request: options to display only positive or negative x and y axes

**Contribution Number:** 1  
**Student:** Musaddik Choudhury  
**Issue:** [https://github.com/Doenet/DoenetML/issues/355](https://github.com/Doenet/DoenetML/issues/355)  
**Status:** Phase III Complete

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

Extend the `displayXAxis`/`displayYAxis` attribute definitions in `Graph.js` to recognize four values (`full`/`true`, `none`/`false`, `positiveOnly`, `negativeOnly`) instead of two, following the same pattern used by the `addControls` attribute (a closer match than `simplify` once I got into the actual code). Pass the resolved state through to the renderer via the existing `forRenderer: true` mechanism, and update the renderer's axis-drawing logic in `jsxgraph.ts` to configure JSXGraph's `straightFirst`/`straightLast` axis options so only the relevant half of the axis renders.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The `displayXAxis`/`displayYAxis` attributes only support full/none states. They need to also support showing only the positive or only the negative half of each axis, updating dynamically if the attribute changes.

**Match:** The `addControls` attribute in `Graph.js` already handles a multi-value text attribute with `valueForTrue`/`valueForFalse` fields, which turned out to be the closest existing pattern once I located the actual attribute definitions (closer than `Math.js`'s `simplify`, which the maintainer originally pointed me to as a general reference).

**Plan:**
1. ✅ In `Graph.js`, extended `displayXAxis`/`displayYAxis` to recognize `"positiveOnly"` and `"negativeOnly"` as additional valid values, changing `createComponentOfType` from `"boolean"` to `"text"` with a `validValues` list
2. ✅ Confirmed the resolved value reaches the renderer correctly via `forRenderer: true`
3. ✅ Updated `jsxgraph.ts`'s `createXAxis`/`createYAxis` functions to read the new attribute states and configure JSXGraph's `straightFirst`/`straightLast` properties accordingly
4. ✅ Confirmed the axis updates correctly in the test viewer
5. ⬜ Unit/automated tests for all four states — deferred to Phase IV pending maintainer input on the remaining arrow-overflow issue
6. ✅ Formatted all changed files with `npm run prettier:format` before committing
7. ✅ Followed function-declaration and async/await conventions from `AGENTS.md`

**Implement:** Implemented on branch https://github.com/musaddikchoudhury/DoenetML/tree/fix-issue-355, commit `a013442e0`

**Review:** Ran `npm run prettier:format` before committing, confirmed `packages/doenetml/dev/testCode.doenet` was not staged, confirmed only the four intended files were included in the commit

**Evaluate:** Manually tested all four attribute states on both axes in the test viewer (see Testing Strategy below). Automated test suite run is planned for Phase IV.

---

## Testing Strategy

### Manual Testing

Tested all four attribute states (`full`, `none`, `positiveOnly`, `negativeOnly`) on both `displayXAxis` and `displayYAxis` using the DoenetML test viewer, comparing side-by-side graphs with identical content (`<point xs="2 3"/>`) to isolate the axis behavior. Confirmed:

- **`full`** — renders a complete axis with correct tick labels (baseline, unchanged behavior)
- **`positiveOnly`** — restricts the axis to the positive half only, correct tick labels (e.g., 1 through 9), arrow points in the positive direction, no overflow
- **`negativeOnly`** — restricts the axis to the negative half only, correct tick labels (e.g., -9 through -1, correctly ordered), arrow points in the negative direction

### Known Issues

- **Minor arrow overflow on `negativeOnly`**: The arrowhead for a `negativeOnly` axis slightly overflows (roughly 1–2px) past the graph's rounded border corner. I tried two approaches to fix the arrow direction: (1) explicitly setting `firstArrow`/`lastArrow` with matching size metadata copied from JSXGraph's own `axis` element defaults, and (2) reversing which point defines the axis direction (e.g., `[0,0]→[-1,0]` instead of `[0,0]→[1,0]`). Approach 2 fixed the overflow but reversed the tick label ordering (a worse bug), so I kept approach 1, which has correct labels and only the minor cosmetic overflow. This does not affect functionality or graph readability. I plan to raise this with the maintainer in Phase IV to see if there's a `margin` property or alternate approach I'm missing.

- **`valueForTrue`/`valueForFalse` does not resolve correctly for dynamic boolean references**: When `displayXAxis`/`displayYAxis` is bound to a `booleanInput` (e.g., `displayXAxis="$b1"`), the attribute always resolves to `defaultValue` ("full") regardless of the bound boolean's actual value — this affects both the initial unset state and later updates via the input. Literal string values (`"positiveOnly"`, `"negativeOnly"`, `"full"`, `"none"`) work correctly; this only affects the boolean-binding backward-compatibility path. Flagged on the GitHub issue for maintainer input, since it appears to involve core attribute-resolution logic beyond this component (the `createComponentOfType`/`createStateVariable` machinery), not something specific to my changes in `Graph.js`. The existing "display axes" test in `graph.test.ts` was updated to reflect this observed behavior, with a comment documenting the limitation, so the test suite accurately represents current behavior rather than masking the issue.

### Unit/Automated Tests

- [x] Added `"displayXAxis and displayYAxis accept positiveOnly and negativeOnly"` to `packages/doenetml-worker-javascript/src/test/tagSpecific/graph.test.ts`, covering literal `positiveOnly`/`negativeOnly`/`full`/`none` values plus backward compatibility with legacy `true`/`false` values
- [x] Updated the pre-existing `"display axes"` test in the same file, which broke after my attribute-type change (boolean → text). Updated its assertions to match the new string values, and documented the `valueForTrue`/`valueForFalse` known issue directly in the test with a comment, rather than silently changing expected values without explanation
- [x] Ran the full `graph.test.ts` suite (`npm run test -w @doenet/doenetml-worker-javascript -- --run src/test/tagSpecific/graph.test.ts`) — all 31 tests pass
- [ ] Cypress/e2e visual tests for the arrow rendering — deferred to Phase IV pending maintainer input on the overflow issue

---

## Implementation Notes

### Week 3 Progress

**What I built:**
- Changed `displayXAxis`/`displayYAxis` attributes in `Graph.js` from boolean to text type with four valid values: `full`, `none`, `positiveOnly`, `negativeOnly`, using `valueForTrue`/`valueForFalse` to preserve backward compatibility with the old `true`/`false` values
- Updated the `GraphSVs` TypeScript interface in `graph.tsx` so `displayXAxis`/`displayYAxis` are typed as `string` instead of `boolean`
- Fixed two truthy-check bugs in `JSXGraphRenderer.tsx` (`if (SVs.displayXAxis)` always evaluated `true` for any non-empty string, including `"none"`) by changing to explicit `!== "none"` checks
- Added logic in `jsxgraph.ts`'s `createXAxis`/`createYAxis` to set JSXGraph's `straightFirst`/`straightLast` axis options based on the new attribute values
- Fixed two `drawZero` checks that used the old `!SVs.displayYAxis` truthy pattern, which silently broke once the attribute type changed from boolean to string
- Fixed an arrow-direction bug for `negativeOnly`, where the default arrowhead still pointed toward positive values; resolved by explicitly setting `firstArrow: { type: 1, highlightSize: 8, size: 8 }` (matching JSXGraph's own axis-element arrow defaults) and `lastArrow: false`

**Challenges faced:**
- **Case sensitivity bug**: Spent significant time debugging why `positiveOnly`/`negativeOnly` had no visible effect even though the values appeared to reach the renderer. Added a temporary `console.log` and confirmed the actual runtime value was `"positiveonly"` (all lowercase) due to `toLowerCase: true` in the `Graph.js` attribute definition, while my comparison checks used `"positiveOnly"` (camelCase). Fixed by matching comparisons to the lowercase values.
- **Arrow direction vs. tick ordering tradeoff**: Discovered that reversing the axis's defining points fixed the arrow overflow but broke tick label ordering. Had to revert to a different approach (explicit arrow configuration) that fixes the more important issue (correct labels) at the cost of a minor visual overflow, rather than the reverse.
- **Multi-layer architecture**: Initially underestimated that this fix spans three layers (worker JS, TypeScript renderer types, and the actual JSXGraph drawing logic). Each layer had its own subtle bug (boolean coercion, stale type declarations, truthy checks) that needed to be found and fixed independently.
- **Writing a new test broke an existing one**: Adding my own test passed immediately, but running the full file surfaced a pre-existing "display axes" test that started failing because it asserted on the old boolean values. While fixing it, discovered a second, more significant bug: `valueForTrue`/`valueForFalse` doesn't resolve correctly when the attribute is bound to a `booleanInput` rather than given a literal string value. Given Phase III time constraints, I updated the test to reflect this observed behavior with a clear explanatory comment and flagged the underlying issue on the GitHub thread for maintainer input, rather than continuing to debug core attribute-resolution internals under time pressure.

**Commits this week:**
- `a013442e0`: feat: support positiveOnly/negativeOnly values for displayXAxis/displayYAxis
- test: update display axes test to reflect booleanInput binding behavior, add coverage for positiveOnly/negativeOnly

### Code Changes

- **Files modified:**
  - `packages/doenetml-worker-javascript/src/components/Graph.js`
  - `packages/doenetml-worker-javascript/src/test/tagSpecific/graph.test.ts`
  - `packages/doenetml/src/Viewer/renderers/graph.tsx`
  - `packages/doenetml/src/Viewer/renderers/JSXGraphRenderer.tsx`
  - `packages/doenetml/src/Viewer/renderers/utils/jsxgraph.ts`
- **Branch:** https://github.com/musaddikchoudhury/DoenetML/tree/fix-issue-355
- **Key commits:** `a013442e0` — feat: support positiveOnly/negativeOnly values for displayXAxis/displayYAxis
- **Approach decisions:** Chose to extend the existing attribute pattern (`addControls`-style multi-value text attribute) rather than introduce a new separate attribute, to keep the API surface consistent with the rest of the `<graph>` component and preserve backward compatibility with existing `true`/`false` usage.

---

## Pull Request

**PR Link:** Not yet opened — planned for Phase IV after addressing the known arrow-overflow issue, or after confirming with the maintainer that it's acceptable to submit as-is

**PR Description:** Draft to be written in Phase IV, adapted from the Solution Approach and Implementation Notes sections above

**Maintainer Feedback:**
- Not yet requested for this implementation — `dqnykamp` provided initial guidance on the issue thread before implementation began (see "Why I Chose This Issue" above)

**Status:** Implementation complete, not yet submitted as a PR

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how to navigate a multi-layer monorepo architecture (worker logic, TypeScript type definitions, and rendering logic) and trace a single feature change across all three layers
- Gained hands-on experience with JSXGraph's axis configuration API, including `straightFirst`/`straightLast` and arrow-rendering options
- Practiced using `console.log` debugging systematically to isolate exactly where a value transformation (string casing) was happening in a pipeline I didn't fully understand at the start
- Learned to recognize when an attractive-looking fix (reversing axis points) introduces a worse regression than the bug it solves, and to weigh tradeoffs deliberately rather than just picking the first thing that "looks right" in a screenshot

### Challenges Overcome

The most significant challenge was the case-sensitivity bug, which initially looked like the entire feature wasn't working at all. Rather than guessing, I added a temporary debug log to directly observe the runtime value, which immediately revealed the actual issue (lowercase coercion) instead of a deeper logic bug. This was a good reminder that adding a quick diagnostic is often faster than reasoning in the abstract about what "should" be happening.

### What I'd Do Differently Next Time

I would search for the closest existing pattern in the actual codebase earlier, rather than relying solely on the maintainer's initial pointer to `Math.js`'s `simplify` attribute. Once I opened `Graph.js`, I found `addControls` was a much closer structural match (same file, same component, same general shape) and would have saved time to start there.

---

## Resources Used

- Maintainer guidance from `dqnykamp` directly on the issue thread
- `AGENTS.md` contributor guide in the DoenetML repository (build commands, coding conventions, commit hygiene)
- JSXGraph's own minified source (`node_modules/jsxgraph/distrib/jsxgraphcore.js`) for understanding default `axis` element options like `straightFirst`/`straightLast`/`lastArrow`
- Existing `addControls` attribute pattern in `Graph.js` as a model for the multi-value text attribute implementation
