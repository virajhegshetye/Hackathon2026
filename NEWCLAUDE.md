You're all set up — CLAUDE.md is already there and Claude Code auto-loaded it (you can see `~\Workspace\BAMOE` as the working directory). Ignore the `/modernisation-toolkit` commands for now — that's a generic plugin, not needed for what we planned. Just type plain instructions at the `>` prompt.

**Step 1 — press Escape to clear that `/`, then type this first:**

```
Read CLAUDE.md fully before doing anything else. Confirm you understand the folder structure, the current migration scope, and the completeness check process.
```

Press Enter. Wait for it to respond confirming it read the file.

**Step 2 — run the completeness check (don't skip to migration yet):**

```
Do the Completeness Check from CLAUDE.md for both endpoints listed in Current Migration Scope. Also read application-dev.properties in rhpam-jdk8-old for scheduler-related config as described. Reference https://www.ibm.com/docs/en/ibamoe/9.4.x if the local PDF doesn't cover something. Don't migrate or write any code yet — just report your findings as a list.
```

Press Enter. This will take a bit — it's grepping through both folders, reading the PDF, and possibly checking the IBM docs site.

**Step 3 — review what it reports.** It should give you:
- Full class list for both endpoints
- Anything "MISSED" that wasn't in your original list
- Any shared classes to be careful with
- Scheduler config properties found + what's missing on BAMOE side

Paste me that output (or a screenshot) if you want a second opinion before proceeding.

**Step 4 — only after you're happy with the list, say:**

```
Proceed with migrating class by class, starting with SchedulerInfoService and SchedulerInfoServiceImpl. Show me the diff before moving to the next class.
```

**One important note:** your header shows `haiku-default` as the active model — that's a lighter/faster model, not ideal for this kind of careful multi-file migration work. Before Step 1, run:

```
/model
```

and switch to Sonnet or Opus for this task — Haiku is more likely to miss dependency chains or misjudge scope, which is exactly the risk you're trying to avoid here.