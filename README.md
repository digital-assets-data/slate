DAD API Docs (Slate)
----

See slate docs at [slate's repo](https://github.com/slatedocs/slate)

## Deploying
 * from root, run ```bundle exec middleman build --clean```
 * commit/push/merge your build (we commit the output of the build step to avoid having to mess with ruby during the web build - that build is already slow enough)
 * this repo is a git submodule of the `web` repo, which is where the documentation is hosted, so:
    * set the `slate` submodule in `web` to track the build commit from above
    * deploy web
    * profit.
