
- Create a `GITHUB_TOKEN` with repo scope and add it to the project's environment variables in Circle-CI

- Before running autotag for the first time you need to establish an initial semver tag, eg: `v0.0.1`

```console
git tag -m'initial tag' v0.0.1
git push --tags
```

- deleting a tag

```console
git fetch --tags
git tag -d TAG
git push origin :refs/tags/TAG
```

Making of the orb
-----------------

- pantheon-systems or joemiller org? pantheon-systems might be better.
  - users need to trust the code since it runs in their delivery pipeline
  - let's try to do it with pantheon

orb design:

1. most common use cases:

    ```yaml
    orbs:
      release: TODO/TODO@0.0.1
    ```

    Libraries:
    ```yaml
    workflows:
      version: 2
      primary:
        jobs:
          - test
          - autorelease/release-library:
              requires:
                - test
              filters:
                branches:
                  only:
                    - master
    ```

    Binaries:
    ```yaml
    workflows:
      version: 2
      primary:
        jobs:
          - test
          - autorelease/release-binaries:
              requires:
                - test
              filters:
                branches:
                  only:
                    - master
    ```

Parameters?

all jobs require `$GITHUB_TOKEN` env var that can create/update releases on the github repo

- jobs:
  - release-library:
    - autotag_version
    - hub_version
    - executor:  use alternate executor, default: circleci/golang:1.12
    - changelog_file: if set, use this instead of generating one

  - release-binaries:
    - executor:  use alternate executor, default: circleci/golang:1.12
    - changelog_file: if set, use this instead of generating one
    - disable_docker: default false. if set, skip the setup-remote-docker step
    - autotag_version
    - hub_version
    - goreleaser_version

- steps:
  - install-autotag
    - autotag_version
  - install-hub
    - hub_version
  - generate-changelog
  - release-tag
  - release-binaries
    - goreleaser_version


**lifecycle hooks?** https://circleci.com/orbs/registry/orb/circleci/docker-publish#orb-source

- after_checkout
- before_release
- after_release

```yaml
after_checkout:
  description: Optional steps to run after checking out the code.
  type: steps
  default: []

 steps:
  - checkout
  - when:
      name: Run after_checkout lifecycle hook steps.
      condition: << parameters.after_checkout >>
      steps: << parameters.after_checkout >>

jobs:
  - docker-publish/publish:
      after_checkout:
        - run:
            name: Do this after checkout.
            command: echo "Did this after checkout"
```

**conditional example**:

```yaml
parameters:
  deploy:
    description: Whether or not to push image to a registry.
    type: boolean
    default: true

steps:
  - when:
    - name: run deploy
    - condition:  <<parameters.deploy>>
    - steps:
      - run: deploy.sh
```

**how to test?**

https://github.com/CircleCI-Public/docker-orb/blob/master/.circleci/config.yml
https://github.com/CircleCI-Public/orb-tools-orb

starter kit - https://github.com/CircleCI-Public/orb-starter-kit
https://circleci.com/blog/creating-automated-build-test-and-deploy-workflows-for-orbs/
https://circleci.com/blog/creating-automated-build-test-and-deploy-workflows-for-orbs-part-2/

circle looks like they use jobs to kickoff other workflows .. but it looks hella complicated

> Crucially, because the git tag is attached to the same commit as the initial one we pushed from our local machine, this new CircleCI build is seen as part of the same set of VCS status checks. That is, for a particular pull request, both sets of workflow jobs will be attached to the same commit, making it easy for developers to evaluate a given code change.

approach:
- use the circle orb starter kit and follow their pattern for lint/pack/integration tests. use [destructured yaml format][destructured-yaml-format]
- should be able to test the install steps, the changelog steps, etc..
- might have to skip testing the release steps for now.
- do it under personal joemiller namespace for now, if we can

[destructured-yaml-format]: https://circleci.com/docs/2.0/local-cli/#packing-a-config


### Build out notes

Tried the `orb-init.sh` script but ended up having to do a lot of manual steps instead:

1. create github repo: pantheon-systems/go-autoreleaser-orb
1. follow project on circleci
1. setup circle config:

    ```console
    mv config.yml .circleci/config.yml
    sed -i -e 's/<orb-namespace>/pantheon-systems/g' .circleci/config.yml
    sed -i -e 's/<orb-name>/go-autoreleaser/g' .circleci/config.yml
    ```

1. Create orb and test publish the first `dev:alpha` tag:

    ```console
    circleci orb create pantheon-systems/go-autoreleaser

    # test publish the hello world orb
    circleci config pack src > orb.yml
    circleci orb publish orb.yml pantheon-systems/go-autoreleaser@dev:alpha
    ```

1. Create CircleCI API Token, add to the env vars on the project's CircleCI page as `CIRCLE_TOKEN`
1. Create an SSH key to allow CircleCI to push tags to Github:

    ```console
    ssh-keygen -t rsa -b 4096 -m PEM -N "" -f circle-ssh-key
    ```

1. Add the private key (`circle-ssh-key`) to CircleCI under the project's SSH keys page
   (https://circleci.com/gh/pantheon-systems/go-autoreleaser-orb/edit#ssh).
   Set the hostname to `github.com`
1. Add the public key (`circle-ssh-key.pub`) to Github as a new deploy key with write access
   (https://github.com/pantheon-systems/go-autoreleaser-orb/settings/keys)
