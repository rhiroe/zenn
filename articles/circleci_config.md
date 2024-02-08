---
title: Rails + PostgreSQL なアプリの RSpec + Rubocop な CircileCI を回す
date: 2019-8-23
tags: ["Rails", "CircleCI"]
excerpt: 備忘録
---

## 前置き
タイトルの通りRailsアプリのテストをCircleCIでやろうとしたらハマったのでメモする。
```bash
$ ruby -v
ruby 2.6.3p62 (2019-04-16 revision 67580) [x86_64-darwin18]
$ rails -v
Rails 6.0.0
```

## 最終的なconfig.yml
```yaml
# Ruby CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-ruby/ for more details
#
version: 2.1
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: circleci/ruby:2.6.3-node-browsers
        environment:
          BUNDLER_VERSION: 2.0.1
          RAILS_ENV: test
          PGHOST: 127.0.0.1
          PGUSER: postgres

      # Specify service dependencies here if necessary
      # CircleCI maintains a library of pre-built images
      # documented at https://circleci.com/docs/2.0/circleci-images/
      - image: circleci/postgres:11.3

    working_directory: ~/repo

    steps:
      - checkout

      # Setup bundler 2+
      - run:
          name: setup bundler
          command: |
            gem update --system
            gem uninstall bundler
            rm /usr/local/bin/bundle
            rm /usr/local/bin/bundler
            gem install bundler

      # Download and cache dependencies
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "Gemfile.lock" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-

      - run:
          name: install dependencies
          command: |
            bundle install --jobs=4 --retry=3 --path vendor/bundle

      - save_cache:
          paths:
            - ./vendor/bundle
          key: v1-dependencies-{{ checksum "Gemfile.lock" }}

      # Database setup
      - run: bundle exec rake db:create db:schema:load

      # run tests!
      - run:
          name: run tests
          command: |
            mkdir /tmp/test-results
            TEST_FILES="$(circleci tests glob "spec/**/*_spec.rb" | \
              circleci tests split --split-by=timings)"
    
            bundle exec rspec \
              --format progress \
              --format RspecJunitFormatter \
              --out /tmp/test-results/rspec.xml \
              --format progress \
              $TEST_FILES

      # collect reports
      - store_test_results:
          path: /tmp/test-results
      - store_artifacts:
          path: /tmp/test-results
          destination: test-results

      # run rubocop!
      - run:
          name: Rubocop
          command: bundle exec rubocop
```

## ハマってから解決までの流れ
### 最初はSample .ymlを少しいじっただけでビルドしてみた

RubyとPostgreSQLのイメージのタグを変更したくらい

```yaml
# Ruby CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-ruby/ for more details
#
version: 2
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: circleci/ruby:2.6.3-node-browsers

      # Specify service dependencies here if necessary
      # CircleCI maintains a library of pre-built images
      # documented at https://circleci.com/docs/2.0/circleci-images/
      # - image: circleci/postgres:11.3

    working_directory: ~/repo

    steps:
      - checkout

      # Download and cache dependencies
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "Gemfile.lock" }}
            # fallback to using the latest cache if no exact match is found
            - v1-dependencies-

      - run:
          name: install dependencies
          command: |
            bundle install --jobs=4 --retry=3 --path vendor/bundle

      - save_cache:
          paths:
            - ./vendor/bundle
          key: v1-dependencies-{{ checksum "Gemfile.lock" }}

      # Database setup
      - run: bundle exec rake db:create
      - run: bundle exec rake db:schema:load

      # run tests!
      - run:
          name: run tests
          command: |
            mkdir /tmp/test-results
            TEST_FILES="$(circleci tests glob "spec/**/*_spec.rb" | \
              circleci tests split --split-by=timings)"

            bundle exec rspec \
              --format progress \
              --format RspecJunitFormatter \
              --out /tmp/test-results/rspec.xml \
              --format progress \
              $TEST_FILES

      # collect reports
      - store_test_results:
          path: /tmp/test-results
      - store_artifacts:
          path: /tmp/test-results
          destination: test-results
          
      # run rubocop!
      - run:
          name: Rubocop
          command: bundle exec rubocop
```

### bundlerのバージョンがおかしいと怒られる

```bash
#!/bin/bash -eo pipefail
bundle install --jobs=4 --retry=3 --path vendor/bundle
Traceback (most recent call last):
	2: from /usr/local/bin/bundle:23:in `<main>'
	1: from /usr/local/lib/ruby/2.6.0/rubygems.rb:302:in `activate_bin_path'
/usr/local/lib/ruby/2.6.0/rubygems.rb:283:in `find_spec_for_exe': Could not find 'bundler' (2.1.0.pre.1) required by your /home/circleci/repo/Gemfile.lock. (Gem::GemNotFoundException)
To update to the latest version installed on your system, run `bundle update --bundler`.
To install the missing version, run `gem install bundler:2.0.1`
Exited with code 1
```

### bundler2系をインストールしてそっちを使うように追記した

```yaml
# ...

        environment:
          BUNDLER_VERSION: 2.0.1
# ...

      # Setup bundler 2+
      - run:
          name: setup bundler
          command: |
            gem update --system
            gem uninstall bundler
            rm /usr/local/bin/bundle
            rm /usr/local/bin/bundler
            gem install bundler

# ...
```

### RAILS_ENV=developmentになってるしDBに接続できないと怒られる
```bash
#!/bin/bash -eo pipefail
bundle exec rake db:create db:schema:load
RAILS_ENV=development environment is not defined in config/webpacker.yml, falling back to production environment
could not connect to server: No such file or directory
	Is the server running locally and accepting
	connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
Couldn't create 'app_development' database. Please check your configuration.
rake aborted!
ActiveRecord::NoDatabaseError: could not connect to server: No such file or directory
	Is the server running locally and accepting
	connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
...
```

### RAILS_ENVにtestを指定し、HOSTも指定した
```yaml
# ...

        environment:
          
          # ...
          
          RAILS_ENV: test
          PGHOST: 127.0.0.1

# ...
```

### DBを作成する権限がないと怒られる
```bash
#!/bin/bash -eo pipefail
bundle exec rake db:create db:schema:load
RAILS_ENV=test environment is not defined in config/webpacker.yml, falling back to production environment
FATAL:  role "circleci" does not exist
Couldn't create 'app_test' database. Please check your configuration.
rake aborted!
PG::ConnectionBad: FATAL:  role "circleci" does not exist
```

### USERにpostgresを指定してやる
```yaml
# ...

        environment:
          
          # ...
          
          PGUSER: postgres

# ...
```

### rspec_junit_formatterがないとrspecに怒られる
```bash
#!/bin/bash -eo pipefail
mkdir /tmp/test-results
TEST_FILES="$(circleci tests glob "spec/**/*_spec.rb" | \
  circleci tests split --split-by=timings)"

bundle exec rspec \
  --format progress \
  --format RspecJunitFormatter \
  --out /tmp/test-results/rspec.xml \
  --format progress \
  $TEST_FILES
Requested historical based timing, but they are not present.  Falling back to name based sorting
bundler: failed to load command: rspec (/home/circleci/repo/vendor/bundle/ruby/2.6.0/bin/rspec)
LoadError: cannot load such file -- rspec_junit_formatter
```

### rspec_junit_formatter gemを追加する
```ruby
# Gemfile
# ...

group :development, :test do
  
  # ...
  
  gem 'rspec_junit_formatter'
end

# ...
```

### Rubyのバージョンがおかしいとrubocopに怒られる
```bash
#!/bin/bash -eo pipefail
bundle exec rubocop
Error: RuboCop found unsupported Ruby version 2.2 in `TargetRubyVersion` parameter (in vendor/bundle/ruby/2.6.0/gems/bootsnap-1.4.4/.rubocop.yml). 2.2-compatible analysis was dropped after version 0.69.
Supported versions: 2.3, 2.4, 2.5, 2.6, 2.7
Exited with code 2
```

...なんで！？

### バージョンを指定して、vendor配下を対象外にする
```yaml
# .rubocop.yml
# ...

AllCops:
  TargetRubyVersion: 2.6.3
  Exclude:
    - vendor/**/*

# ...
```

これでなんとか動くようになった...。
