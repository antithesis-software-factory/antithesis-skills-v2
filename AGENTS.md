# AGENTS.md

This repo packages the Antithesis skills as one plugin, `plugins/antithesis`, for each agent platform.

## Development-only content

Some guidance exists only to support our own evaluation tooling and must not ship. Put it in its own file whose first line is `<!-- DEV-ONLY: ... -->`, and reference it from other files only inside a `<!-- dev-only:start -->` … `<!-- dev-only:end -->` block. `scripts/strip-dev-only` removes both before a release, and `scripts/strip-dev-only --check` fails while any remain.
