# Build time improvements.

Travis CI use branching and project workflows to enhance the quality, speed and feedback loops for your software assets, these are just some of the ways you can increase the speed and quality even more-so. 

## Use Travis CI cache and reduce clone depth to speed up setup time

Don't forget about `fast_finish` and clone depth to speed times up, here's a quick example: 

```yaml
 matrix:
   fast_finish: true
   include:
```


## Simple ways to speed the Travis build up 

One core functionality of Bash is to manage parameters. A parameter is an entity that stores values and is referenced by a name, a number or a special symbol. Parameters referenced by a name are called variables (this also applies to arrays), so for example in a `before_script: ${!PREFIX*}` (in your `.travis.yml` file). 

## Try strings instead of single-element lists

One of the great things about Travis is, Travis will accept `strings` instead of single-element lists for most fields:

Before:

```python
language: python
services:
  - postgresql

before_install:
  - pip install --upgrade pip
install:
  - pip install -r requirements/dev.txt
script:
  - coverage run -m pytest
```

Travis will the below `.travis.yml` as the top `.travis.yml`:

```python
language: python
services: postgresql

before_install: pip install --upgrade pip
install: pip install -r requirements/dev.txt
script: coverage run -m pytest
```

## Multi-line string syntax
Travis utilizes YAML's multi-line string, this means the syntax can be used for short inline scripts, in this scenario we'll use the `after_script:` example:

```yaml
after_script:
   - |
    lockfile="puppet/$ENVIRONEMT/Puppetfile.lock"
    if [[ $(git status -s "$lockfile") ]]; then
      echo "The librarian-puppet lockfile ($lockfile) has changes."
      echo 'Please run `bundle exec rake module_install` and commit the changes.'
      exit 1
    fi
```

## Docker image caching 

Let's say you're building docker images, it is desirable to cache image layers between builds. Docker images are built from Dockerfiles, and each instruction in a Dockerfile defines a layer of the image to be built.

Again, there are a few different ways to achieve this with Travis CI. I recommend using Docker’s `--cache-from` option. This allows you to specify an existing and trusted image, in turn making the build go faster, here's a quick example:

```yaml
dist: xenial
services:
  - docker
language: bash
env:
  global:
    - IMAGE_TAG=praekeltorg/mysite
    - REGISTRY_USER=praekeltorg-automation
    - secret: <REGISTRY_PASS encrypted>

before_script:
  - docker pull "$IMAGE_TAG" || true
script:
  - docker build --pull --cache-from "$IMAGE_TAG" -t "$IMAGE_TAG" .

before_deploy:
  - echo -n "$REGISTRY_PASS" | docker login -u "$REGISTRY_USER" --password-stdin
deploy:
  provider: script
  script: docker push "$IMAGE_TAG"
  on:
    branch: master
 ```
 
## Release via build stages

So take a look at the following `.travis.yml` file:

```yaml
dist: xenial
language: python
cache: pip

before_install:
  - pip install --upgrade pip
install:
  - pip install -r requirements/dev.txt
  - pip install codecov
script:
  - coverage run -m pytest
after_success:
  - codecov
  
matrix:
  include:
    - python: '3.7'
    - python: '3.6'
    - python: '3.5'
```

What you want to pay attention to is the following: 

```yaml
matrix:
  include:
    - python: '3.7'
    - python: '3.6'
    - python: '3.5'
 ```
 
 We only want to publish once, this can now be acheived via:
 
  ```yaml
  - stage: release
      if: tag IS present
      deploy:
       provider: pypi
        user: praekelt.org
        password:
          secure: <encrypted password>
        distributions: sdist bdist_wheel
        on:
          tags: true
      install: skip
     script: skip
     after_success: skip
  ```

## Boolean expressions 

Tests and boolean expressions can be used to include/exclude certain steps on specific builds. For example, the below code snippet doesn’t run the mix inch.report command if the Elixir version is 1.6, so these conditionals also can be important: 

```yaml
matrix:
  include:
    - elixir: '1.6'
    - elixir: '1.7'
    - elixir: '1.8'

script:
  - mix test
  # Inch 2.0 only works on Elixir >= 1.7 # This is just an example
  - '[[ "$TRAVIS_ELIXIR_VERSION" = 1.6 ]] || mix inch.report'
  - mix format --check-formatted'
```

## Running semantic-releases only

This example is a minimal configuration for a semantic-release with a build running Go 1.6 and 1.7. This `.traivs.yml` creates a release build stage that runs semantic-release only after all test jobs are successful. It's recommended to run the semantic-release command in the Travis deploy step so if an error occurs the build will fail and Travis will send a notification.

Note: It's not recommended to run the semantic-release command in the Travis script step as each script in this step will be executed regardless of the outcome of the previous one. 

A Running the tests in the script step of the release stage is not necessary as the previous stage(s) already ran them. This in turn will ncrease the speed of the build and the script step of the release stage can be overwritten.

```yaml
language: go

go:
  - 1.6
  - 1.7

jobs:
  include:
    # Define the release stage that runs semantic-release
    - stage: release
      # Advanced: optionally overwrite your default `script` step to skip the tests
      # script:
      #   - make (or something different)
      deploy:
        provider: script
        skip_cleanup: true
        script:
          # Use nvm to install and use the Node LTS version (nvm is installed on all Travis images)
          - nvm install lts/*
          - npx semantic-release
   ```
