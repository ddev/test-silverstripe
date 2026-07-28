# test-silverstripe for DDEV testing only

This repo is a fixture consumed by `ddev/ddev`'s own test suite
(`pkg/ddevapp/ddevapp_test.go`, the Silverstripe case — search for the
`Name: "TestPkgSilverstripe"` marker rather than trusting any particular
array index/line number, since that shifts as other test cases are
added/removed). That test downloads a tagged GitHub release of this repo's
two assets and expects, in a fresh `ddev` silverstripe project:

- `SourceURL` → `silverstripe-base.tar.gz` → the code, vendor, and
  vendor-exposed public assets, tarred from repo root
- `DBTarURL` → `db.tar.gz` → a `db.sql` dump, tarred
- `DynamicURI: "/"` → response body contains a generator meta tag whose
  version suffix matches whatever `silverstripe/framework` version is
  actually installed, e.g. `<meta name="generator" content="Silverstripe
  CMS 6.2">` — see gotcha below, **do not hardcode this**.

There's no `FilesTarballURL`/upload-dir check for this fixture (unlike the
sibling `ddev/test-magento2` fixture) — no media/asset upload testing
happens here.

When you cut a new release here, `ddev/ddev`'s test file (`SourceURL`,
`DBTarURL`, `DynamicURI.Expect`) needs to be updated to match — that's a
separate PR in `ddev/ddev`, not this repo.

## Current versions (as of this fixture's last rebuild)

- PHP: 8.3 (`composer.json` constraint is `^8.3`)
- `silverstripe/recipe-cms`: 6.2.0
- `silverstripe/framework`: 6.2.2
- `silverstripe/cms`: 6.2.1
- Theme: `silverstripe/startup-theme` 1.0.12 (composer-managed, not vendored
  into git — `.gitignore` excludes `/themes/startup-theme/`)

Check Packagist for whatever's actually latest before assuming these are
still current — this fixture tracks the relaxed `^6` constraint on
`recipe-cms`, so a plain `composer update` will drift forward over time.

## To recreate

1. Clone this repository.
2. Spin up DDEV (there is intentionally no `.ddev/` committed — `.gitignore`
   covers it, and it must never be included in the tarball):

   ```bash
   ddev config --project-type=silverstripe --docroot=public
   ddev start
   ```

3. Update the vendor libraries:

   ```bash
   ddev composer update
   ```

   No `repo.magento.com`-style marketplace credentials needed — all of
   Silverstripe's packages are public on Packagist.

4. Check what actually got installed, since the generator meta tag must
   match exactly:

   ```bash
   ddev exec composer show -s | grep -A1 '^name'
   grep -o 'silverstripe/framework": "[^"]*"' composer.lock
   ```

   **Gotcha:** `<meta name="generator" content="Silverstripe CMS X.Y">` is
   computed at runtime from whatever `silverstripe/framework` version is
   actually installed (`SiteTree::getGenerator()` in
   `silverstripe/silverstripe-cms`, `code/Model/SiteTree.php` — it takes the
   major.minor of the installed framework version via
   `preg_match('#^([0-9]+\.[0-9]+)\.[0-9]+$#', ...)`). If `composer update`
   landed on e.g. `6.3.x`, the new `ddev/ddev` test expectation is
   `Silverstripe CMS 6.3`, not whatever this README says above.

5. Build the database and confirm the site works:

   ```bash
   ddev sake dev/build flush=all
   ddev launch /admin   # confirm login works (admin / password, from .env)
   ```

   DDEV's silverstripe apptype populates `.env` with the DB connection vars
   (`SS_DATABASE_*`) and `MAILER_DSN` automatically on `ddev start`/
   `ddev config` — don't hand-edit those unless you're intentionally
   changing the fixture's admin credentials.

6. Expose vendor assets into `public/_resources` and build the code tarball:

   ```bash
   ddev composer vendor-expose
   tar -czf silverstripe-base.tar.gz --exclude=.ddev --exclude=.git .
   ```

   `public/_resources` is populated with relative symlinks back into
   `vendor/` and `themes/` (not copies) — that's fine for this fixture since
   both directories are tarred up together and the symlinks stay relative.
   Double-check this hasn't changed with `tar -tvzf silverstripe-base.tar.gz
   | grep '^l'` — an absolute-path symlink would break on extraction
   elsewhere (this bit the sibling Magento fixture, whose static-asset
   symlinks are absolute).

7. Export the database and build the db tarball:

   ```bash
   ddev export-db --gzip=false --file=db.sql
   tar -czf db.tar.gz db.sql
   ```

8. Create a release tagged to match the installed `silverstripe/framework`
   version (e.g. `6.2.2`) and upload both assets (`gh` CLI required):

   ```bash
   gh release create 6.2.2 --title 6.2.2 --notes-file notes.txt
   gh release upload 6.2.2 silverstripe-base.tar.gz db.tar.gz
   ```

9. Update `ddev/ddev`'s `pkg/ddevapp/ddevapp_test.go` Silverstripe case
   (`SourceURL`, `DBTarURL`, `DynamicURI.Expect`) to point at the new tag and
   generator version — confirm who's driving that PR before doing it
   yourself, it's a separate repo.

## About Silverstripe

Silverstripe is a New Zealand based organisation, and primary developer of
the Open Source CMS and Framework of the same name, with a focus on the CMS
user and developer flexibility. It is largely used in governmental
environments and has users in New Zealand, Australia, Canada, the
Netherlands, Germany and Poland, amongst others.

For questions about Silverstripe specifically, join the [Silverstripe Users
Slack channel](https://www.silverstripe.org/community/slack-signup/). Do not
ask questions about Silverstripe here!

## @todo

Add a .github workflow that automatically uploads the artifacts, instead of
relying on a manual method.
