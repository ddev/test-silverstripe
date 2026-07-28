# CLAUDE.md

Guidance for a future Claude session asked to update this repo again.

## What this repo is

A normal, git-tracked Silverstripe CMS composer project whose sole purpose
is to be tarred up and released as a fixture for `ddev/ddev`'s own test
suite (`pkg/ddevapp/ddevapp_test.go`, the Silverstripe case — search for the
`Name: "TestPkgSilverstripe"` marker, don't trust a hardcoded line/index
number, the array shifts as other test cases are added/removed).

The full step-by-step recreate procedure lives in `README.md` — keep it
current; this file is about judgment calls and pitfalls, not the mechanical
steps.

## Cross-repo dependency

Cutting a new release here does nothing for `ddev/ddev` until its test file
is updated to point at the new tag (`SourceURL`, `DBTarURL`,
`DynamicURI.Expect` for the Silverstripe case). That's a separate PR in a
separate repo — don't assume it's this repo's job unless asked.

## Non-obvious things learned doing this in 2026 (recipe-cms 6.1.x → 6.2.0)

- **The generator meta tag version is computed at runtime, never hardcode
  it.** `<meta name="generator" content="Silverstripe CMS X.Y">` is derived
  from whichever `silverstripe/framework` version is actually installed
  (`SiteTree::getGenerator()` in `silverstripe/silverstripe-cms`,
  `code/Model/SiteTree.php`, taking the major.minor via
  `preg_match('#^([0-9]+\.[0-9]+)\.[0-9]+$#', ...)`). After any
  `composer update`, check `composer.lock` (or `composer show -s`) for the
  actual installed `silverstripe/framework` version and use *that*
  major.minor when updating `ddev/ddev`'s test expectation.
- **`composer.json` uses a relaxed `^6` constraint on `recipe-cms`, so this
  fixture drifts forward on its own.** Don't trust any version number
  written in this repo's docs (including this file) as still current —
  check Packagist for `silverstripe/recipe-cms` before doing anything.
- **DDEV's `silverstripe` apptype writes to `.env` itself**, adding
  `SS_DATABASE_NAME`, `SS_DATABASE_PORT`, `SS_DATABASE_CLASS`, and
  `MAILER_DSN` on `ddev config`/`ddev start`. Seeing `.env` show up as
  modified after a routine `ddev start` is expected DDEV behavior, not
  something to investigate as unexpected drift — check what changed with
  `git diff .env` and commit it along with the version bump.
- **`public/_resources` is populated with *relative* symlinks** back into
  `vendor/` and `themes/startup-theme/` (via `composer vendor-expose`'s
  `auto` method, which prefers symlinks over copies on Unix). This is
  unlike the sibling `ddev/test-magento2` fixture, where Magento's
  static-asset publisher creates *absolute*-path symlinks that only work
  when re-extracted at the exact same container path. Here the symlinks are
  fine to tar up as-is — verified with `tar -tvzf silverstripe-base.tar.gz |
  grep '^l'` after every rebuild, since this is cheap to check and a
  regression here (e.g. from a future vendor-plugin change) would silently
  break extraction elsewhere.
- **The login form's actual POST target is
  `/Security/login/default/LoginForm`, not `/Security/login`.** Posting
  directly to `/Security/login` just re-renders the login page with no
  visible error (not even a validation message) — it silently no-ops rather
  than failing loudly. If scripting a login check (e.g. via curl, since
  there's no browser available in this environment), read the actual
  `action=` attribute out of the rendered form rather than assuming the
  page URL doubles as the POST target. Also required: the hidden
  `AuthenticationMethod` field and the exact-case `action_doLogin` submit
  button name (not `action_dologin`).
- **No `repo.magento.com`-style marketplace credentials needed** — all of
  Silverstripe's packages are public on Packagist, which simplifies the
  update flow a lot compared to the Magento fixture (no `auth.json`
  wrangling, no `ddev composer create-project` empty-root gotcha since this
  is an in-place version bump, not a from-scratch reinstall).
- **Never commit or release directly without asking.** Committing,
  opening/merging PRs, creating GitHub releases, and uploading release
  assets are all shared, visible, and not-trivially-reversible — confirm
  with the user first every time, don't assume a prior green light carries
  forward to the next update cycle.
- **Never use inline heredocs in Bash tool commands for multi-line content**
  (`git commit -F`, `gh pr create --body-file`, `gh release create
  --notes-file`, etc.) — write the content to a file first and pass the
  file path. Inline heredocs in this kind of harness can both hard-fail
  (`unexpected EOF`) and silently hang forever.
- **Actually test things** (spin up DDEV, hit the URLs, check the tarball
  contents) rather than reasoning about whether they'll work. This is what
  caught both the relative-vs-absolute symlink question and the
  `/Security/login` vs `/Security/login/default/LoginForm` login-form
  gotcha above.
