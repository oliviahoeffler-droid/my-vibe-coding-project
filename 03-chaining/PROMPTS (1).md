# PROMPTS.md: Living Prompt Pack

> Module 3 · Prompt Chaining. Re-architect the build with prompt chains; capture the reusable ones here.

## How to use this pack

_Each prompt is a reusable step. Chain them: the output of one becomes the input to the next._

## Prompt chain: Check-out hardening chain

### Step 1: Expand: add the two missing screens
```
Build the next phase of this app in a strict sequence:
1. Add a screen Rep calendar connection screen — reps exist in Settings, but there's no screen where a rep connects their calendar or sees their own day. Even a placeholder "Connect Google Calendar" per rep would close the loop.
2. Navigation: write the logic so Rep calendar connection screen exists under the Settings page
3. Add a screen Speed-to-lead stats reporting screen/feature — the core promise (response in under 10 seconds) has no reporting view: average time-to-first-text, booking rate, leads by status over time.
4. Add a Lost-lead recovery screen/feature — leads stuck at "no answer" have no re-engagement path (no retry call/text action or a follow-up queue screen).

Build these in order so Settings is the anchor for Rep Calendar connection screen. Speed-to-lead status and lost-lead recovery can be anchored on to the "leads" screen as a module, or a new screen.
```

### Step 2: Behavior: handle loading, empty and error states
```
Apply the following logic constraints to the {{screen}} flow:
- Use skeleton screens for the {{list}} loading state.
- If no data is present, show the empty state: "{{empty-state message}}".
- On fetch failure, trigger the error state: "{{error message}}".

Maintain the same design language throughout and tether all behavior strictly to these rules.
```

### Step 3: Refine: polish one thing on one screen
```
The {{screen}} needs a professional {{style}} polish.
1. Start by listing the 3 biggest gaps in typography and spacing compared to {{reference}}.
2. Once you've identified those, resize the headers and update the {{element}} to match.

Don't change anything else in the project or touch the underlying logic.
```

## Reusable techniques learned

- Keep refining before digging in to adding data hookups, or you'll kill your credits
- Have the AI tell you what's missing and suggest what's next; verify

## What broke (and the fix)

_Where a single mega-prompt failed and chaining fixed it._

Nothing broke for me. Chaining allowed me to focus on small parts over time that needed improvement.
