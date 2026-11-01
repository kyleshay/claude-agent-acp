# @toyon-dev/claude-agent-acp

This fork exists only to ship upstream PRs that have not landed yet. Toyon's daemon
pins it instead of `@agentclientprotocol/claude-agent-acp` until they do.

## Shape: a queue, not a product

- One branch per change, and each one is an upstream PR (ours or someone else's).
  `pr/1350-steer-priority` is `pull/1350/head` fetched from `origin`. No commit goes
  on a release branch without an upstream PR number.
- One release branch per upstream tag: branch from the tag, cherry-pick every `pr/*`
  branch, then add one commit that renames the package to `@toyon-dev/claude-agent-acp`
  and sets the version to `<tag>-toyon.1`. `bin` stays `claude-agent-acp`, so the
  daemon's `bin` field never changes.
- If upstream rejects a patch, drop it within one release and move the workaround
  into the daemon. Once every queued PR has landed upstream, the fork has no reason
  to exist and the pin goes back to upstream.
- Nothing else goes on the fork until it has been filed upstream. Next candidates,
  in order: the fold notification from upstream issue #1341
  (`_session/steering_folded {uuid}` when the steered echo lands), then a PreToolUse
  `in_progress` start signal.

Queued now:

| Branch                   | Upstream PR | What it does                                                              |
| ------------------------ | ----------- | ------------------------------------------------------------------------- |
| `pr/1350-steer-priority` | #1350       | `_meta.steering.priority` (`now`, `next`, `later`) on `_session/steering` |

The upstream workflows (`ci.yml`, `publish.yml`, `conventional-prs.yml`) only fire
on `main`, so they never run on the fork's branches. Leave them as they are.

## Per-release routine

Remotes: `origin` is upstream, `fork` is github.com/kyleshay/claude-agent-acp.

```sh
git fetch origin main --tags
git fetch origin pull/1350/head:pr/1350-steer-priority   # refresh each queued PR
git checkout -b toyon-release vX.Y.Z                     # the new upstream tag
git cherry-pick pr/1350-steer-priority                   # every pr/* branch
# rename: name @toyon-dev/claude-agent-acp, version X.Y.Z-toyon.1,
# in package.json and the root of package-lock.json; the recorded ACP scenarios
# carry agentInfo.name, so rename it there too, then commit:
grep -rl '"@agentclientprotocol/claude-agent-acp"' src/tests \
  | xargs sed -i '' 's#"@agentclientprotocol/claude-agent-acp"#"@toyon-dev/claude-agent-acp"#g'
npm ci && npm run check && npm run build && npm run test:run
npm pack --dry-run
grep -n 'steering.priority\|STEER_PRIORITIES' dist/acp-agent.js
npm publish --access public --tag latest   # npm requires a tag for a -toyon.N prerelease
git push fork toyon-release
```

The grep must find the priority read in the built output before publishing. Run the
tests outside any sandbox that blocks file watchers: `session-index.test.ts` relies
on them, and fails all at once without them. Then
move the daemon pin in Toyon (`packages/daemon/src/agent/registry.ts`) to the new
version.
