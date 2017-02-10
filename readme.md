# Automation TLDR

These are the incantations for setting up a Heroku app that automatically
builds your project on a schedule and publishes it to npm and GitHub.

A detailed version of this automation technique can be found at
[zeke.sikelianos.com/npm-and-github-automation-with-heroku](http://zeke.sikelianos.com/npm-and-github-automation-with-heroku/)

```sh
# Create release script
mkdir -p scripts
touch scripts/release.sh
chmod +x scripts/release.sh

# Set up `npm run release`
npm i -g npe
npe "scripts.release" "node scripts/release.sh"

# Create Heroku app
heroku create
heroku buildpacks:add -i 1 https://github.com/zeke/github-buildpack
heroku buildpacks:add -i 2 https://github.com/zeke/npm-buildpack
heroku buildpacks:add -i 3 heroku/nodejs
heroku config:set GITHUB_AUTH_TOKEN=YOUR_GITHUB_TOKEN
heroku config:set NPM_AUTH_TOKEN=YOUR_NPM_TOKEN
heroku config:set NPM_CONFIG_PRODUCTION=false
heroku addons:create scheduler
heroku addons:open scheduler

# Sha. Push it.
git commit -am "add release script"
git push heroku master

heroku run npm run release
```

Content for `release.sh`

```sh
#!/usr/bin/env bash

# set -x            # print commands before execution
set -o errexit    # always exit on error
set -o pipefail   # honor exit codes when piping
set -o nounset    # fail on unset variables

repo=$npm_package_repository_url
repo="${repo/git+/}"
repo="${repo/.git/}"
project=$(basename $repo)

git clone $repo $project
cd $project
npm install
npm run build
npm test
[[ `git status --porcelain` ]] || exit
git add .
git config user.email $npm_package_author_email
git config user.name $npm_package_author_name
git commit -am "update $npm_package_name"
npm version minor -m "bump minor to %s"
npm publish
git push origin master --follow-tags
```
