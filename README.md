# Heroku Buildpack for Node.js & Phantomjs

This is a fork of the official [Heroku buildpack](http://devcenter.heroku.com/articles/buildpacks) for Node.js apps that also installs [Phantomjs](http://phantomjs.org/) at compile time. The scripts for installing Phantom were taken directly from Beedesk's [custom Phantomjs buildpack](https://github.com/beedesk/heroku-buildpack-phantomjs).


## How it Works

Apps are built via one of four paths:

1. A regular `npm install` (first build; default scenario)
2. Copy existing `node_modules` from cache, then `npm prune`, then `npm install` (subsequent builds)
3. Skip dependencies (if package.json doesn't exist but server.js does)
4. Skip cache, run `npm rebuild` before `npm install` (`node_modules` are checked into source control)

You should only use #3 (omitting package.json) for quick tests or experiments.

You should never use #4 - it's included for backwards-compatibility and will generate warnings.
**Checking in `node_modules` is an antipattern.**
For more information, see [the npm docs](https://docs.npmjs.com/misc/faq#should-i-check-my-node_modules-folder-into-git-)

For technical details, check out the [heavily-commented compile script](https://github.com/heroku/heroku-buildpack-nodejs/blob/master/bin/compile).

## Documentation

For more information about using Node.js and buildpacks on Heroku, see these Dev Center articles:

- [Heroku Node.js Support](https://devcenter.heroku.com/articles/nodejs-support)
- [Getting Started with Node.js on Heroku](https://devcenter.heroku.com/articles/nodejs)
- [10 Habits of a Happy Node Hacker](https://blog.heroku.com/archives/2014/3/11/node-habits)
- [Buildpacks](https://devcenter.heroku.com/articles/buildpacks)
- [Buildpack API](https://devcenter.heroku.com/articles/buildpack-api)


## Legacy Compatibility

For most Node.js apps this buildpack should work just fine.
If, however, you're unable to deploy using this new version of the buildpack, you can get your app working again by locking it to the previous version:

```
heroku config:set BUILDPACK_URL=https://github.com/heroku/heroku-buildpack-nodejs#v63 -a my-app
git commit -am "empty" --allow-empty
git push heroku master
```

Then please open a support ticket at [help.heroku.com](https://help.heroku.com/) so we can diagnose and get your app running on the default buildpack.

## Options

### Specify a node version

Set engines.node in package.json to the semver range
(or specific version) of node you'd like to use.
(It's a good idea to make this the same version you use during development)

```json
"engines": {
  "node": "0.11.x"
}
```

```json
"engines": {
  "node": "0.10.33"
}
```

Default: the
[latest stable version.](http://semver.io/node)

### Specify an npm version

Set engines.npm in package.json to the semver range
(or specific version) of npm you'd like to use.
(It's a good idea to make this the same version you use during development)

Since 'npm 2' shipped several major bugfixes, you might try:

```json
"engines": {
  "npm": "2.x"
}
```

```json
"engines": {
  "npm": "^2.1.0"
}
```

Default: the version of npm bundled with your node install (varies).

### Enable or disable node_modules caching

For a 'clean' build without using any cached node modules:

```shell
heroku config:set NODE_MODULES_CACHE=false
git commit -am 'rebuild' --allow-empty
git push heroku master
heroku config:unset NODE_MODULES_CACHE
```

Caching node_modules between builds dramatically speeds up build times.
However, `npm install` doesn't automatically update already-installed modules
as long as they fall within acceptable semver ranges,
which can lead to outdated modules.

Default: `NODE_MODULES_CACHE` defaults to true

### Enable or disable devDependencies installation

During local development, `npm install` installs all dependencies
and all devDependencies (test frameworks, build tools, etc).
This is usually something you want to avoid in production, so
npm has a 'production' config that can be set through the environment:

To install *dependencies only:*

```shell
heroku config:set NPM_CONFIG_PRODUCTION=true
```

To install *dependencies and devDependencies:*

```shell
heroku config:set NPM_CONFIG_PRODUCTION=false
```

Default: `NPM_CONFIG_PRODUCTION` defaults to true on Heroku

### Configure npm with .npmrc

Sometimes, a project needs custom npm behavior to set up proxies,
use a different registry, etc. For such behavior,
just include an `.npmrc` file in the root of your project:

```
# .npmrc
registry = 'https://custom-registry.com/'
```

### Reasonable defaults for concurrency

This buildpack adds two environment variables: `WEB_MEMORY` and `WEB_CONCURRENCY`.
You can set either of them, but if unset the buildpack will fill them with reasonable defaults.

- `WEB_MEMORY`: expected memory use by each node process (in MB, default: 512)
- `WEB_CONCURRENCY`: recommended number of processes to Cluster based on the current environment

Clustering is not done automatically; concurrency should be part of the app,
usually via a library like [throng](https://github.com/hunterloftis/throng).
Apps without any clustering mechanism will remain unaffected by these variables.

This behavior allows your app to automatically take advantage of larger containers.
The default settings will cluster
1 process on a 1X dyno, 2 processes on a 2X dyno, and 12 processes on a PX dyno.

For example, when your app starts:

```
app[web.1]: Detected 1024 MB available memory, 512 MB limit per process (WEB_MEMORY)
app[web.1]: Recommending WEB_CONCURRENCY=2
app[web.1]:
app[web.1]: > example-concurrency@1.0.0 start /app
app[web.1]: > node server.js
app[web.1]: Listening on 51118
app[web.1]: Listening on 51118
```

Notice that on a 2X dyno, the
[example concurrency app](https://github.com/heroku-examples/node-concurrency)
listens on two processes concurrently.

### Chain Node with multiple buildpacks

This buildpack automatically exports node, npm, and any node_modules binaries
into the `$PATH` for easy use in subsequent buildpacks.

## Feedback

Having trouble? Dig it? Feature request?

- [help.heroku.com](https://help.heroku.com/)
- [@hunterloftis](http://twitter.com/hunterloftis)
- [github issues](https://github.com/heroku/heroku-buildpack-nodejs/issues)

## Hacking

To make changes to this buildpack, fork it on Github. Push up changes to your fork, then create a new Heroku app to test it, or configure an existing app to use your buildpack:

```
# Create a new Heroku app that uses your buildpack
heroku create --buildpack <your-github-url>

# Configure an existing Heroku app to use your buildpack
heroku config:set BUILDPACK_URL=<your-github-url>

# You can also use a git branch!
heroku config:set BUILDPACK_URL=<your-github-url>#your-branch
```

## Testing

The buildpack tests use [Docker](https://www.docker.com/) to simulate
Heroku's Cedar and Cedar-14 containers.

To run the test suite:

```
test/docker
```

Or to just test in cedar or cedar-14:

```
test/docker cedar
test/docker cedar-14
```

The tests are run via the vendored [shunit2](http://shunit2.googlecode.com/svn/trunk/source/2.1/doc/shunit2.html)
test framework.
