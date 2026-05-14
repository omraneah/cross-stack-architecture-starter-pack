# Governance

## What this repo is

A reference set of architectural boundaries. The author maintains it; readers use it however serves them. There is no organization to enforce against, no compliance to opt into. The repo is a thinking tool, not a policy document.

## What "non-negotiable" means here

Within this pack: the *principle* of each boundary is non-negotiable. The judgment section names trade-offs honestly; alternatives are valid; the principle itself stands.

In your team or codebase: the boundaries are reference, not policy. You decide which to apply, in what order, and where the trade-offs land for your context.

## Doctrine vs boundary

The boundaries describe principles that hold across operators. The doctrine (`DOCTRINE.md`) is one operator's specific picks within those principles. Treat the doctrine as a worked example of how one set of choices plays out; not as a recommendation to copy.

## How the repo evolves

Pull requests that are welcome:

- Sharpen a boundary's wording — a clearer principle, a more honest judgment, a more specific signal of violation.
- Name a failure mode the author missed.
- Add a worked example to `examples/` for a stack not yet covered.
- Correct a misstatement, a typo, or a broken cross-reference.

Pull requests that are not the right fit:

- Embedding a specific framework's syntax inside a boundary (those go to `examples/`).
- Converting the doctrine into "best practice" — the doctrine is one operator's choices, not consensus.
- Adding boundaries for areas the author hasn't operated (see README *Limitations*).

The author makes the final call on what's a principle (boundary) vs an opinion (doctrine) and ships changes accordingly.
