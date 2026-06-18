# Complete Action

> ⚠️ This action **commits, merges to `main`, and pushes to origin**. Per
> [`context/ai-interaction.md`](../../../../context/ai-interaction.md) ("ask before
> committing; ask to delete the branch"), you MUST get explicit confirmation first.

0. **Confirm before doing anything.** Ensure `npm run build` passes, then show the user:
   - changed files (`git status`) + a short diff summary,
   - the proposed commit message,
   - the plan: merge into **main**, delete the branch, and **push to origin**.
   Ask: "Proceed with commit + merge to main + push? (yes/no)" — do **not** continue
   until the user explicitly says yes. If the build fails, stop and fix first.
1. Stage all changes and commit with a descriptive message
2. Switch to main and merge the feature branch (no push yet)
3. Delete the local feature branch
4. Reset current-features.md:
   - Change H1 back to `# Current Feature`
   - Clear Goals and Notes sections (keep placeholder comments)
   - Add feature summary to the END of History
5. Commit the reset: `chore: reset current-features.md after completing [feature]`
6. Push main to origin ONCE (single push with all changes)
7. If feature branch was previously pushed, delete it from origin