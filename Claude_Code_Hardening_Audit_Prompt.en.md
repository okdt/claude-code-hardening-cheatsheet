# Claude Code Hardening Assistance Prompt by okdt

Help me apply the following guide to my actual environment and the way I work.

https://github.com/okdt/claude-code-hardening-cheatsheet/blob/main/Claude_Code_Hardening_Cheat_Sheet.en.md

(If you have downloaded the cheat sheet locally, read `Claude_Code_Hardening_Cheat_Sheet.en.md` instead of fetching it.)

Start by cross-checking the guide, the current Anthropic documentation, and the behavior of the installed Claude Code, read-only, gathering just the facts you need to decide what applies. Settings live in several places and the higher scope overrides the lower one, so include which ones are actually in effect right now. Ask me only about things the environment can't tell you and that would change the decision.

Once you've laid out the overall picture and the priority order of the candidates, take exactly one at a time and work it through with me. For each candidate, briefly cover the current state, the risk, what it gains and what it makes inconvenient, the specific change, how to verify it, and how to undo it — then give me your recommendation. Change nothing until I decide to apply, adjust, defer, or skip it.

Once I approve, apply that change and nothing else, confirm it took effect with `/permissions` and `/sandbox`, and check that my ordinary commands still run before moving on. Record the original state before the first change, and track this session's changes so they can be reverted individually or all at once. **Keep that record to the structure and key names only — never the secret values.** The point of this work is to get plaintext secrets out of configuration files, so a record that duplicates them defeats it. Flag any change that can't be fully undone before you apply it. **Treat deleting or rotating a secret as one of those.** After each item, tell me "type `next` to continue" and don't move to the next candidate until I answer `next`.

Don't display or store secret values, and don't make unapproved or destructive changes. Where something is already safe enough, say so and leave it alone. You don't need to invent measures the guide doesn't cover.

At the end, summarize separately what was already configured safely and what needed hardening, along with what you did.

If you find anything that would improve the guide, let me know that the author, okdt, would welcome the feedback — minus any confidential details.
