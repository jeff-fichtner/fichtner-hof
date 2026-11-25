# fichtner-hof

The family vacation house's website. Nothing is built. `ROADMAP.md` is the assessment of the current
WordPress.com site and the migration plan; the decisions in its section 4 gate the first step, and
none is made yet. The site's shape (deploy mode and render, from snackbyte-base) is chosen when the
project is picked up, after section 4's first question. Listed under Someday in snackbyte-ops'
FOCUS.md: worked only on Jeff's word.

- **Personal details:** the roadmap names the house's street address, a choice Jeff made knowingly
  when the repo went public. Add no other personal detail (names with phone numbers or email
  addresses, guests' details) to anything committed here.
- **Spec engine:** `snackbyte-speckit-engine` (deprecated; its successor is not published yet),
  installed with `specify extension add --dev`, local plug. A `--dev` re-install writes the
  `.claude/skills/speckit-*` mirrors back as symlinks into a gitignored folder. Replace them
  with files again right after any re-install:
  `find .claude/skills -type l -exec sh -c 'cp "$1" "$1.tmp" && mv "$1.tmp" "$1"' sh '{}' ';'`
