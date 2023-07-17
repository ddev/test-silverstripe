# Silverstripe DDEV test repository

This repository is for the DDEV automated testing, and should not be used as a base for a real project. (Although it should work just fine).

## Usage and creation of assets/artifacts

To create the artifacts needed for DDEV automated tests, the following steps need to be taken:

1. Clone this repository.
2. Run `composer install` to install the vendor related libraries.
    1. If you want to update this repository with the latest and greatest libraries available, run `composer update`.
3. Run `composer vendor-expose` to copy the assets (css and javascripts) to the public folder.
4. Create a tar.gz from the resulting files in the folder where you cloned the repository to.
    1. Easiest is to run `tar -czf silverstripe-base.tar.gz .`
    2. WARNING: Do _not_ include a potential `.ddev` folder! If you are using DDEV to create the artifacts, ensure the folder is not included
5. Ensure you have a functioning environment to build your database (May we suggest using DDEV?).
6. Build the database with `vendor/bin/sake dev/build flush=all`.
7. Export the database and tar.gz it (`tar -czf db.tar.gz db.sql`).

You now have all the artifacts needed to update the Test repository.

Update the repository and release, with the artifacts you created.


## Ensure it all works

Update `ddev/pkg/ddevapps/ddevapp_test.go` at the 17th (Silverstripe) test configuration, and point the SourceURL and DBTarURL to the correct location.

Run your tests for Silverstripe:

`GOTEST_SHORT=17 make testpkg TESTARGS="-run TestDdevFullSiteSetup"`

If there are any failures, you can either fix the issues yourself, or get in touch with the team maintaining DDEV. (In case of Silverstripe, contact @Firesphere)

## About Silverstripe

Silverstripe is a New Zealand based organisation, and primary developer of the Open Source CMS and Framework of the same name, with a focus on the CMS user and developer flexibility. It is largely used in governmental environments and has users in New Zealand, Australia, Canada, the Netherlands, Germany and Poland, amongst others.

For questions about Silverstripe specifically, join the [Silverstripe Users Slack channel](https://www.silverstripe.org/community/slack-signup/). Do not ask questions about Silverstripe here!

# @todo

Add a .github workflow that automatically uploads the artifacts, instead of relying on a manual method.

### Notes

DDEV automated tests are run against Silverstripe CMS 5. Silverstripe CMS 4 works just as well, however, it's Mailer settings require a custom YML configuration.
For more information on using DDEV and Silverstripe's email capturing, see https://firesphere.dev/articles/ddevelopment-environment/