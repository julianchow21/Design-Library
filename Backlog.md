# Backlog

Intake queue for the pattern library. One line per idea, newest on top, delete the line once intaken (see CLAUDE.md, Floors not ceilings).

- Natural-language quick entry, free-text command bar parsed into typed records with live preview and syntax-help popover, Manpower Portal `index.html:4097`, parked 16/07/2026, high effort to generalise beyond leave-tracking vocabulary
- Statement reconciliation gate, penny-perfect balance check before any write with abort on mismatch and dedupe on merge, Spending `tools/parse_statements.py:122`, parked 16/07/2026, Python pipeline doctrine, systems doc not a card
- Designed Empty State upgrade, Journal v3.1 pairs the stored card's named-state/why/one-action formula with a muted brand mark (`whaleMark`, 52px, opacity .55) on the true first-run state only, never repeated on other empty states, source Journal `index.html` no-pages empty state, added 16/07/2026, upgrade candidate for the stored card
- Same-source duplicate collisions, the concurrent 16/07/2026 harvests intook the same Collectibles patterns twice (Recoverable Trash vs Trash Flow, Numeric Filter vs Filter Search), one of each pair should be folded into the other, Julian to pick survivors
- Twin-intake overlap sweep, the Journal and Collectibles intakes landed sibling cards on 16/07/2026 (Modal Shell vs Modal Stack, CSV Import vs Paste Import, Undo Snapshot vs Undo Stack, Offline Shell.md vs App Updates.md), kept both sides with cross-references, Julian to decide merge or keep-both per pair
- Sync conflict policy fork, three variants now live, starter keeps dirty local edits over cloud (`index.html:578`), Collectibles discards dirty local when cloud is strictly newer with a logged toast (`app.js:435`), Journal reconciles by timestamp at table level (`Sync Engine.md`), added 16/07/2026, Julian to pick one house policy, then align the others
- Toast upgrade for the starter, Collectibles toast (icons, animated progress-bar dismiss, popover API, `app.js:4246`) beats the starter's plain `toast()`, added 16/07/2026, promotion candidate since the pattern class already ships in the starter
- Refresh queue, rate-limited self-healing fetch queue (daily credit cap, resume after interruption, error classification), Collectibles `app.js:7797`, parked 16/07/2026, thin UI surface, revisit when a second app needs scheduled API refresh
- Custom chart builder, drag fields onto axis pills with saved and pinned charts, Collectibles `app.js:9822`, parked 16/07/2026, heavy rebuild, take when a second app needs charts
- AI chat panel shell, typing indicator plus data-snapshot prompt build plus inline charts in replies, Collectibles `app.js:6334`, parked 16/07/2026, take the shell only, strip domain prompts
- Pipeline stepper, advance-a-stage status stepper with a split-row modal, Collectibles `features.js:1870`, parked 16/07/2026, needs de-domaining from the eBay flow
