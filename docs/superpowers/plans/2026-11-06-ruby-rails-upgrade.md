# Ruby 4.0.9 / Rails 8.1.4.1 Upgrade Across the Internal Bullet Train Forks — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Use the `rails-cto:rails-upgrade` method (selective merge, never `rails app:update`) for every Rails bump, and `bullet-train:link-core` / `bin/hack` from `tujucreative/bullet_train-claude_code` when testing core against the starter.

**Goal:** Every `tujucreative/bullet_train*` Ruby repo runs green on Ruby 4.0.9 and Rails 8.1.4.1, resolves Bullet Train gems from our forks instead of upstream, and builds from our own container images. That gives us one baseline for the internal tools and the Malaysia/ASEAN work.

**Architecture:** First upgrade the fork everything else depends on (`bullet_train-core`), then the starter and the demo site that consume it, then the Pro plugins in dependency order (billing → billing-usage → billing-stripe / umbrella_subscriptions; action_models, audit_logs and conversations are independent). Each repo is its own task, on branch `claude/nice-brown-i7hj5r`, with its own green test run. Repoint every upstream reference (RubyGems, `bullet-train-pro`, `bullet-train-co`, `ghcr.io/bullet-train-co`) to `tujucreative` so we stop depending on upstream release timing.

**Tech Stack:** Ruby 4.0.9, Rails 8.1.4.1, the Bundler version that ships with Ruby 4.0.9, PostgreSQL + Redis (starter), SQLite (plugin dummy apps), Node 24.15.0, Tailwind CSS 3, GitHub Actions, GHCR.

**Spec:** No separate spec doc. The baseline below (measured 2026-11-06) and the Decisions section are the spec.

### Baseline (measured)

| Repo | Ruby (`.ruby-version`) | Rails (lock) | `bullet_train` (lock) | Notes |
|---|---|---|---|---|
| bullet_train-core | 4.0.3 | via starter | 1.45.1 (source) | CI tests against `bullet-train-co/bullet_train`; images from `docker.io/library/ruby:$RUBY_VERSION-slim` |
| bullet_train (starter) | 4.0.3 | 8.1.4 | 1.45.1 from RubyGems | `load_defaults 8.0` + untouched `new_framework_defaults_8_1.rb`; Dockerfiles `FROM ghcr.io/bullet-train-co/...` |
| bullet_train-demo_site | 4.0.3 | 8.1.3 | 1.45.1 from RubyGems | Same layout as starter |
| bullet_train-billing | none | 7.0.4.2 | 1.14.0 | Bundler 2.3.26 |
| bullet_train-billing-usage | none | 7.0.6 | 1.6.10 | Bundler 2.3.5, last commit 2023-11 |
| bullet_train-billing-stripe | none | 8.0.2.1 | 1.29.0 | Gemfile: `bullet_train-billing` from `bullet-train-pro` |
| bullet_train-billing-umbrella_subscriptions | none | 8.0.1 | 1.17.1 | `required_ruby_version >= 2.6.0` |
| bullet_train-action_models | none | 7.0.4.2 | 1.6.25 | Only plugin with CI (Ruby 3.0, tests commented out) |
| bullet_train-audit_logs | 3.2.1 | 7.0.4.2 | none | Depends on paper_trail, diffy |
| bullet_train-conversations | none | 7.0.8.4 | 1.7.17 | Needs icu4c (see README) |
| bullet_train-mcp | n/a (TypeScript) | n/a | n/a | Consumes the starter's OpenAPI spec |

Latest stable on 2026-11-06: Rails **8.1.4.1** (2026-11-04), Ruby **4.0.9**. Ruby 4.1.0-preview1 exists; Rails has no 8.2 prerelease yet.

### Decisions

1. **Target:** Ruby 4.0.9 and Rails 8.1.4.1, both stable. Do not target Ruby 4.1 until 4.1.0 ships (normally 25 December) and Rails supports it; that becomes a follow-up patch-level plan.
2. **Keep fork divergence small.** Do not change gemspec `required_ruby_version` or `rails` floors, gem `VERSION` constants, or `BULLET_TRAIN_VERSION` (stays `1.45.1`). Only change lockfiles, `.ruby-version`, Gemfile sources, CI and Docker references. That way upstream merges into our forks stay clean.
3. **Gem distribution:** Use Bundler `git` sources on `branch: "main"` of each `tujucreative` fork, with `Gemfile.lock` pinning the SHA. A private gem server (GitHub Packages) is a later decision, made once internal tools need released versions.
4. **Images:** Publish to `ghcr.io/tujucreative/bullet_train/{base,build,dev}:1.45.1`, built from Ruby 4.0.9.
5. **Plugins with dead upstream history** (billing-usage, umbrella_subscriptions): still upgrade them. Retiring them is a product decision for the internal-tools brainstorm, not part of this plan.

## Global Constraints

- Ruby `4.0.9` in every `.ruby-version`; the starter Gemfile keeps `ruby(File.read(... ".ruby-version"))`.
- Rails `8.1.4.1`; starter/demo Gemfile constraint stays `gem "rails", "~> 8.1.0"`.
- `BULLET_TRAIN_VERSION = "1.45.1"` unchanged; Node `24.15.0` unchanged; Tailwind stays v3 (`^3.4.17`). Do not adopt v4 syntax.
- Never run `bin/rails app:update`. Use the `rails-cto:rails-upgrade` selective merge.
- Work on `claude/nice-brown-i7hj5r` in each repo; push with `git push -u origin claude/nice-brown-i7hj5r`.
- Commits: author `Rudzainy <rudzainy@tujucreative.com>`, one short summary line, no trailers or tool branding.
- No upstream URL may remain in Gemfiles, gemspec `homepage`s, workflows or Dockerfiles. The final grep in Task 12 must return nothing.

## Review Focus

1. **Mixed gem sources.** One core gem resolves from RubyGems while the rest come from the fork, so code silently runs upstream gems. Expected: every `bullet_train*` gem shows a `bundler/gems/bullet_train-core-<sha>` path. Pinned by the `bundle info` check in Task 2, step 4 and in plugin procedure P, step 4.
2. **Ruby 4 bundled-gem removals in code paths tests don't load** (generators, rake tasks, Super Scaffolding templates). Expected: eager load and Super Scaffolding both pass. Pinned by Task 2, step 5 (`zeitwerk:check` + eager load) and Task 2, step 6 (super scaffolding suite).
3. **`raise_on_missing_required_finder_order_columns = true`** raising on `.first`/`.last` for a model with no primary key and no `implicit_order_column` (join tables). Expected: no runtime raise. Pinned by the grep audit in Task 3, step 2.
4. **`action_on_path_relative_redirect = :raise`** blowing up on `redirect_to "foo"` (no leading slash) in an untested controller or gem override. Expected: no raise. Pinned by the grep audit in Task 3, step 2.
5. **Production image built on the wrong Ruby** (stale `ghcr.io/bullet-train-co` base, or a tag reused at 4.0.3). Expected: `ruby -v` inside the image prints 4.0.9. Pinned by Task 4, step 3.

---

### Task 1: bullet_train-core — Ruby 4.0.9 and internal CI wiring

**Files:**
- Modify: `bullet_train-core/.ruby-version` → `4.0.9`
- Modify: `bullet_train-core/.github/workflows/_starter_repo_tests.yml:71` → `repository: tujucreative/bullet_train`
- Modify: `bullet_train-core/.github/workflows/docker.yml`, `_build_docker_image.yml`: image names `bullet-train-co/bullet_train/*` → `tujucreative/bullet_train/*`
- Modify: any other `bullet-train-co` reference under `.github/` (find with `grep -rn "bullet-train-co" .github`)

**Interfaces:**
- Produces: `tujucreative/bullet_train-core@main` as the git source every other repo consumes. Gem names and versions are unchanged (`1.45.1`).

- [ ] **Step 1:** Record the baseline. In `bullet_train`, link core (`bullet-train:link-core` skill, pointing `./local/bullet_train-core` at `../bullet_train-core`) and run `bin/rails test` on Ruby 4.0.3. Save the pass/fail counts to the task notes; later failures are compared against these.
- [ ] **Step 2:** Make the file changes above.
- [ ] **Step 3:** Install Ruby 4.0.9 (`rbenv install 4.0.9` or `mise install ruby@4.0.9`) and run `bundle install` in core. Expected: resolves with no errors, and `bundle exec standardrb` is green.
- [ ] **Step 4:** In the starter with core linked (`BULLET_TRAIN_VERSION` unchanged), run `bin/rails test` and `bin/rails test:system` on 4.0.9. Expected: same counts as step 1 or better. Fix any Ruby 4.0.4–4.0.9 breakage in core itself; a change that would also matter upstream gets a one-line note in `CHANGELOG.md` under an `Internal` heading.
- [ ] **Step 5:** Run `grep -rn "bullet-train-co\|bullet-train-pro" .github`. Expected: no output.
- [ ] **Step 6:** Commit `Upgrade to Ruby 4.0.9 and point CI at internal forks`, push, open a PR to `main`, and merge after CI is green. Tasks 2+ consume `main`.

### Task 2: bullet_train (starter) — Ruby 4.0.9, Rails 8.1.4.1, gems from fork

**Files:**
- Modify: `bullet_train/.ruby-version` → `4.0.9`
- Modify: `bullet_train/Gemfile:118-145`: wrap the 17 core gem lines in one git block (below)
- Modify: `bullet_train/Gemfile.lock` (via bundler only)
- Modify: `bullet_train/Dockerfile:24,55`, `bullet_train/dev.Dockerfile:26`: `ghcr.io/bullet-train-co/` → `ghcr.io/tujucreative/`
- Modify: `bullet_train/.github/workflows/*.yml`: any `bullet-train-co` image or repo reference

**Interfaces:**
- Consumes: `tujucreative/bullet_train-core@main` (Task 1).
- Produces: the canonical starter (Ruby 4.0.9, Rails 8.1.4.1) that `bullet_train-demo_site` (Task 11) and new internal tools are generated from.

Gemfile shape (keep the constant so version-bump tooling still works):

```ruby
BULLET_TRAIN_VERSION = "1.45.1"

git "https://github.com/tujucreative/bullet_train-core.git", branch: "main" do
  gem "bullet_train", BULLET_TRAIN_VERSION
  # ...the other 16 existing bullet_train-* lines, unchanged
end
```

- [ ] **Step 1:** Run the `rails-cto:rails-upgrade` analysis for 8.1.4 → 8.1.4.1. Read the 8.1.4.1 release notes (four-segment releases are normally security fixes), check railsdiff for generated-file changes, and run the JS dependency check (`@rails/*`, `@hotwired/*` in `package.json`). Record any action items in the task notes.
- [ ] **Step 2:** Make the `.ruby-version` and Gemfile changes, then run `bundle update rails bullet_train --conservative`, followed by `bundle update --conservative` for anything Bundler reports as blocking. Expected: `Gemfile.lock` shows `rails (8.1.4.1)` and `ruby 4.0.9`.
- [ ] **Step 3:** Update `@rails/*` npm packages to the 8.1.4.1-matching versions if step 1 found any, then run `yarn install && yarn build && yarn build:css`. Expected: exit 0.
- [ ] **Step 4:** Check gem sources: `for g in $(bundle list --name-only | grep ^bullet_train); do bundle info $g --path; done`. Expected: every path contains `bundler/gems/bullet_train-core-`. None may be under `gems/bullet_train-*-1.45.1`.
- [ ] **Step 5:** Eager-load check: `bin/rails zeitwerk:check` and `RAILS_ENV=test bin/rails runner 'Rails.application.eager_load!; puts :ok'`. Expected: `All is good!` and `ok`. A `LoadError` for an ex-default gem (for example `ostruct`, `logger`, `benchmark`) means adding that gem to the Gemfile of the gem whose code requires it. If that is core, the fix goes in core's gemspec via a core PR.
- [ ] **Step 6:** Full suite: `bin/setup`, `bin/rails test`, `bin/rails test:system`, and the super scaffolding suite as `_run_super_scaffolding_tests.yml` runs it. Expected: counts at least as good as the Task 1 baseline, and no new deprecation warnings in `log/test.log` (`grep -c DEPRECATION log/test.log` is no higher than at baseline).
- [ ] **Step 7:** Make the Dockerfile and workflow changes (the images are published in Task 4; CI image jobs will be red until then, which is expected and should be noted in the PR).
- [ ] **Step 8:** Commit `Upgrade to Ruby 4.0.9 and Rails 8.1.4.1 using internal core fork` and push.

### Task 3: bullet_train (starter) — finalize Rails 8.1 framework defaults

**Files:**
- Modify: `bullet_train/config/initializers/new_framework_defaults_8_1.rb` (enable one option at a time, then delete the file)
- Modify: `bullet_train/config/application.rb:14` → `config.load_defaults 8.1`
- Modify: any app or overridden-gem code the audits flag

**Interfaces:**
- Consumes: Task 2 branch state.
- Produces: `load_defaults 8.1` with no pending defaults. This is required before Rails 8.2, which removes the `raise_on_missing_required_finder_order_columns` toggle.

- [ ] **Step 1:** Enable the six options one per cycle, in this order, running `bin/rails test` after each and fixing failures before moving on:
  1. `action_view.render_tracker = :ruby`
  2. `action_view.remove_hidden_field_autocomplete = true`
  3. `active_support.escape_js_separators_in_json = false`
  4. `action_controller.escape_json_responses = false`
  5. `action_controller.action_on_path_relative_redirect = :raise`
  6. `active_record.raise_on_missing_required_finder_order_columns = true`
- [ ] **Step 2:** Run the Review Focus audits on the app and the linked core checkout: `grep -rnE 'redirect_to\s+"[^/h]' app ../bullet_train-core/*/app` (expected: no hits, otherwise make the path absolute or use a route helper), and list the models with no primary key: `bin/rails runner 'ActiveRecord::Base.descendants.reject(&:abstract_class?).select { _1.primary_key.nil? }.each { puts _1 }'`. For each one listed, set `self.implicit_order_column` or add an explicit `order`.
- [ ] **Step 3:** Add a regression test, `test/models/finder_order_test.rb`, that asserts every listed no-PK model responds to `.first` without raising (`assert_nothing_raised { Model.first }`). If step 2 listed none, skip the test and say so in the PR.
- [ ] **Step 4:** Regenerate the OpenAPI document with `bin/rails bullet_train:api:export_openapi_schema` (writes `tmp/openapi/openapi-<timestamp>.yaml`) and diff it against an export taken on the Task 2 commit. Expected: either no diff, or only escaping differences. Note the diff for Task 12.
- [ ] **Step 5:** Set `load_defaults 8.1`, delete the initializer, and run `bin/rails test && bin/rails test:system`. Expected: green.
- [ ] **Step 6:** Commit `Adopt Rails 8.1 framework defaults` and push.

### Task 4: Internal container images on Ruby 4.0.9

**Files:**
- Modify: `bullet_train-core/.github/workflows/docker.yml` / `_build_docker_image.yml` (already repointed in Task 1; set `RUBY_VERSION` from `.ruby-version`)
- Modify: `bullet_train/.github/workflows/docker-bt-internal.yml`: `PREVIEW_IMAGE_NAME: tujucreative/bullet_train`

**Interfaces:**
- Produces: `ghcr.io/tujucreative/bullet_train/base:1.45.1`, `/build:1.45.1` and `/dev:1.45.1`, which the starter Dockerfiles from Task 2 use.

- [ ] **Step 1:** Build locally to prove the Dockerfiles: `docker build -t bt/base -f dockerfiles/base.Dockerfile --build-arg RUBY_VERSION=$(cat .ruby-version) .` in core. Expected: exit 0.
- [ ] **Step 2:** Run the core image workflow through `workflow_dispatch` on `main` (requires `packages: write` on the `tujucreative` org; confirm with the org owner first).
- [ ] **Step 3:** `docker run --rm ghcr.io/tujucreative/bullet_train/base:1.45.1 ruby -v`. Expected: `ruby 4.0.9`.
- [ ] **Step 4:** Re-run the starter's CI on the Task 2 PR. Expected: Docker jobs are green.
- [ ] **Step 5:** Commit any workflow fixes (`Publish internal images on Ruby 4.0.9`) and push.

### Plugin procedure P (used by Tasks 5–10)

Each Pro plugin is a Rails engine with `test/dummy`, `load_defaults Rails::VERSION::STRING.to_f`, SQLite, and Minitest. For plugin `X` with fork dependencies `D`:

1. Record the baseline: on the plugin's current Ruby, run `bin/rails test` (or `bundle exec rake test`) and write down the counts. If it can't boot, record that instead.
2. Create `.ruby-version` with `4.0.9`, or update it if present.
3. Gemfile: add the core git block from Task 2 containing only the core gems the gemspec names, plus one line per fork dependency in `D`: `gem "bullet_train-<dep>", github: "tujucreative/bullet_train-<dep>", branch: "main"`. Replace every `bullet-train-pro`/`bullet-train-co` source.
4. Run `bundle update` (a full update is fine: these are dev lockfiles, not shipped). Expected: `rails (8.1.4.1)`, `bullet_train (1.45.1)`, and core gem paths under `bundler/gems/bullet_train-core-` (same check as Task 2, step 4).
5. Run the suite and fix breakages. Work through them in this order: Ruby 4 removed default gems (add the gem to the **gemspec** when runtime code needs it, or to the Gemfile when only tests do); the Rails 7.0 → 8.1 upgrade-guide items (`rails-cto:rails-upgrade` analysis for the plugin's lock version → 8.1.4.1); and `bullet_train` 1.x → 1.45.1 API changes (read core `CHANGELOG.md` between the lock version and 1.45.1). Re-generate the dummy app's `db/schema.rb` with `bin/rails db:schema:load db:migrate` when migrations changed.
6. Add or replace `.github/workflows/ci.yml`: `ruby/setup-ruby@v1` with `ruby-version-file: .ruby-version`, `bundler-cache: true`, then `bundle exec standardrb` and `bin/rails test`. For action_models, re-enable the commented-out test job instead of adding a new file.
7. Gemspec `homepage`/metadata: `bullet-train-co` → `tujucreative`.
8. `bundle exec standardrb --fix`, rerun the tests (expected: green), commit `Upgrade to Ruby 4.0.9 and Rails 8.1.4.1`, and push.

### Task 5: bullet_train-billing (P, `D` = none; core gems: `bullet_train`)
Do this first; Tasks 6–8 depend on its `main`. Merge its PR before starting Task 6.

### Task 6: bullet_train-billing-usage (P, `D` = billing; core gems: `bullet_train`)
Largest jump (Rails 7.0.6, `bullet_train` 1.6.10, Bundler 2.3.5). If it can't boot even at baseline, record that and continue.

### Task 7: bullet_train-billing-stripe (P, `D` = billing; core gems: `bullet_train`)
Replace `Gemfile:16` (`bullet-train-pro/bullet_train-billing`). Stripe tests must keep using recorded fixtures or VCR only, never live keys.

### Task 8: bullet_train-billing-umbrella_subscriptions (P, `D` = billing, billing-usage; core gems: `bullet_train`)
Needs the Task 6 merge. Leave `required_ruby_version >= 2.6.0` as is (Decision 2).

### Task 9: bullet_train-action_models, bullet_train-audit_logs (P, independent)
action_models: core gems `bullet_train`, `bullet_train-super_scaffolding`, `bullet_train-fields`; check that `csv`/`matrix` stay in the gemspec. audit_logs: no core gems; `bundle update` should lock `paper_trail` 17.x (the latest, needs activerecord ≥ 7.1); the gemspec floor `>= 14.0` stays. Two repos, two PRs, one review gate, because both are small independent engines.

### Task 10: bullet_train-conversations (P, independent; core gems: `bullet_train-super_scaffolding`)
Install `icu4c` per its README before `bundle install`. Check that `extended_email_reply_parser ~> 0.5` loads on Ruby 4; if it doesn't, vendor a patch in `lib/` and leave a TODO linking the upstream issue.

### Task 11: bullet_train-demo_site — follow the starter
Apply Task 2 (steps 2–8) and Task 3 to the demo site; it shares the starter's layout. Start from Rails 8.1.3, so the step 1 analysis covers 8.1.3 → 8.1.4.1. Commit `Upgrade to Ruby 4.0.9 and Rails 8.1.4.1 using internal core fork`.

### Task 12: Close-out — MCP contract check, version reference, upstream audit

**Files:**
- Modify: `bullet_train-claude_code/skills/bullet-train-versions/SKILL.md` (baseline table: Ruby 4.0.9, Rails 8.1.4.1, `load_defaults 8.1`, gems from the `tujucreative/bullet_train-core` git source)
- Test: `bullet_train-mcp/test/` (existing suite)

- [ ] **Step 1:** Point `bullet_train-mcp` at the OpenAPI document regenerated in Task 3, step 4, then run `npm test` and `npm run typecheck`. Expected: green. Any escaping-related diff from Task 3 must not change tool IDs.
- [ ] **Step 2:** Update the versions skill table and its "Sources" section, then commit `Update version reference for Ruby 4.0.9 and Rails 8.1.4.1`.
- [ ] **Step 3:** Upstream audit across all repos: `grep -rnE "bullet-train-(co|pro)/" --include=Gemfile --include=*.gemspec --include=*.yml --include=*Dockerfile /home/user/bullet_train*`. Expected: no output, apart from links in historic `CHANGELOG.md` entries, which aren't in the grep set.
- [ ] **Step 4:** Run `superpowers:verification-before-completion`, then post one summary listing each repo with its PR link and test counts compared with baseline.

---

## Follow-on plans (out of scope here; each needs its own brainstorm → plan)

These are separate subsystems and must not be mixed into the upgrade PRs.

1. **Internal tool factory.** Generate new internal apps from the upgraded starter: a template repo or `rails new -m` script, shared auth (one Devise/SSO?), which Pro plugins each tool gets, and whether to move from Bundler git sources to GitHub Packages.
2. **Malaysia baseline.** Add the `ms` locale alongside `en` (`config/locales/locales.yml`, plus core and plugin locale files), `config.time_zone = "Asia/Kuala_Lumpur"` as the default, MYR in billing/Stripe prices, MY phone formats (phonelib is already present), SST on invoices, and PDPA 2010 data-handling review.
3. **ASEAN expansion.** Per-team locale and time zone (`id`, `th`, `vi`, `fil`, `zh`, `ta`, `km`, `lo`, `my`), per-country currency and tax (SGD/GST, IDR/PPN, THB/VAT, VND, PHP/VAT), number and date formats via `rails-i18n`, data-residency options, and country-specific payment methods (FPX, PayNow, QRIS, PromptPay) through Stripe.
