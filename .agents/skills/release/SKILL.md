---
name: release
description: Release the Antithesis plugin. Use when the user asks to "release the plugin", "cut a release", "bump the version", or gives a version, like "release plugin 0.2.0".
---

# Release the plugin

A release is a pull request that bumps the plugin's version and adds its notes to `CHANGELOG.md`. The user reviews the PR and merges it. Merging releases it: CI sees a version on the default branch with no GitHub release yet, tags the merge commit `vX.Y.Z`, and creates the release with that version's `CHANGELOG.md` section as its notes.

Users install the plugin from the default branch, and Claude Code and Codex update it when its version changes. So merging the PR ships the release to every user. There are no pre-releases.

## 1. Pick and check the version

Get the current version from `plugins/antithesis/plugin.json`. If the user gave a version, strip any leading `v`. Otherwise, pick the next one once you've read the changes in step 2: bump the minor version if any change is breaking, and the patch version otherwise. Say which you picked and why in the PR, so the user can change it in review.

Before changing anything, check:

- The version is `MAJOR.MINOR.PATCH`, and higher than the current version.
- `git tag -l vX.Y.Z` and `gh release view vX.Y.Z` find nothing.
- `git status --porcelain` is empty, and you're on the default branch, up to date with the remote.

If a check fails, tell the user and stop.

## 2. Write the notes

Write the notes from what changed, not from PR descriptions. A description can show an approach that changed in review, and a later PR can change the behavior again.

Find the previous release with `git fetch --tags && git tag -l 'v[0-9]*' --sort=-v:refname | head -1`. Read what users get from the plugin since then:

- What the skill tells agents to do: `git diff vPREV..HEAD -- plugins/antithesis/skills/`.
- How users install it and what it needs: `git diff vPREV..HEAD -- plugins/antithesis/*.json plugins/antithesis/.*-plugin/ .claude-plugin/ .agents/plugins/ README.md docs/`.

Then list the commits with `git log --oneline vPREV..HEAD`. When the diffs don't explain a change, read its commit with `git show <sha>`. Use the PR number, `(#N)` at the end of a squash-merged subject, only to link the entry.

If there is no previous release, write a short summary of what the plugin does instead: its modules, and the platforms it installs on.

Write the notes at the level of modules and what the agent does differently:

- One entry per module or feature. Merge the PRs that changed the same thing into one entry.
- Leave out rewording that doesn't change what the agent does, and internal changes: CI, scripts, this skill, contributor docs.
- Write each entry as the net change since the previous release. A module that's new in this release is one entry, not the PRs that built it. When one change reverses another, leave out both.
- Put breaking changes first, then the rest, largest first. A change is breaking if something that worked with the previous release now fails or needs the user to act: a renamed module or command, a new requirement on the user's setup, a changed harness layout that existing workloads need to adopt. Start each with `**Breaking**:` and say what to do instead.
- Link every PR that contributes to an entry, e.g. `([#7](https://github.com/antithesishq/antithesis-plugin/pull/7))`.

Put the notes at the top of `CHANGELOG.md`, under `# Version X.Y.Z (YYYY-MM-DD)` with today's date. Create the file if it doesn't exist. Then check that `scripts/changelog-section X.Y.Z` prints them: the release workflow uses the same script.

## 3. Bump the version

Run `scripts/set-version X.Y.Z`, then `scripts/set-version --check`. This sets the version in every manifest.

## 4. Open the PR

```bash
git switch -c release/vX.Y.Z
git add CHANGELOG.md plugins/antithesis/plugin.json plugins/antithesis/.claude-plugin/plugin.json plugins/antithesis/.devin-plugin/plugin.json
git commit -m "Release antithesis plugin X.Y.Z"
git push -u origin release/vX.Y.Z
gh pr create --title "Release antithesis plugin X.Y.Z" --body "<body>"
```

If `git status` shows any other change, stop: the release commit holds only the notes and the version.

In the body, say which version this releases and which release the notes start from. If you picked the version yourself, say why.

Don't merge the PR, and don't tag anything. Give the user the PR's URL. When they ask for changes, push them to the same branch. Once they merge it, CI creates the release.
