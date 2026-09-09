# CLAUDE.md

## Merges

**Never merge anything without an explicit instruction from the user.** That covers pull requests, branch
merges and fast-forwarding `main` — here and in every other Annium repository. Open the pull request,
report it, and stop there. Approval for one merge is not approval for the next one.

**Never merge before CI has finished and passed.** Not with `--admin`, not because the checks look slow,
not because a local run was green — and not because the user asked for it in passing. If asked to merge
while CI is pending, wait for it and say so. A local suite is not CI: it runs on one machine, with that
machine's containers, network and clock.
