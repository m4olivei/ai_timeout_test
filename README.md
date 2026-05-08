# AI Timeout Reproducer

A Drupal 11 recipe that reproduces Pantheon's Fastly 59-second between-bytes
edge timeout in pure form, decoupled from any application-specific AI feature.

## Purpose

Long-running AI calls that buffer their full response server-side will be cut
by Fastly at ~59s, surfacing as `499` in nginx logs and
`net::ERR_INCOMPLETE_CHUNKED_ENCODING` (or a truncated 200) in the browser.

This recipe stands up the minimum Drupal artifacts needed to trigger that
condition on a stock site:

- A barebones `ai_assistant_api` entity (no agent, no tools, no orchestration).
- A stock `ai_chatbot` DeepChat block, placed bottom-right on every Olivero
  page, with **streaming disabled** so the response buffers.
- A system prompt that deterministically forces a ~4,500–5,500 word LLM
  response, reliably exceeding 59 seconds of generation time.

LLM provider is `ai_provider_amazeeio`; bring your own amazee.ai API key.

| Environment                | Expected outcome                                                 |
|----------------------------|------------------------------------------------------------------|
| Local DDEV / vanilla       | HTTP 200, full response, 75–95s elapsed                          |
| Pantheon (or any 59s edge) | HTTP 200 truncated, or `ERR_INCOMPLETE_CHUNKED_ENCODING` at ~59s |

## Requirements

- Drupal 11
- amazee.ai account and API key — https://www.amazee.io/

Module dependencies (`drupal/ai`, `drupal/ai_provider_amazeeio`, `drupal/key`)
are declared in `composer.json` and pulled in automatically.

## Install and apply

From a stock Drupal 11 codebase (e.g. `drupal/recommended-project`):

```bash
# 1. Register this repo as a Composer VCS source.
composer config repositories.ai_timeout_test vcs https://github.com/m4olivei/ai_timeout_test

# 2. Pull in the recipe and its module dependencies.
composer require m4olivei/ai_timeout_test:dev-main

# 3. Apply. You'll be prompted for your amazee.ai API key and LLM host.
drush recipe recipes/ai_timeout_test

# 4. Rebuild caches.
drush cr
```

## Reproduce

1. Log in as an authenticated or administrator user.
2. Open the floating chat in the bottom-right of any page.
3. Send any prompt, e.g. *"Explain how distributed cache invalidation works."*
4. Watch the response in DevTools → Network. On Pantheon (or any host with a
   59s edge cut), expect truncation or `ERR_INCOMPLETE_CHUNKED_ENCODING` right
   around the 59s mark.

## Uninstall

Recipes don't un-apply atomically. To remove cleanly:

```bash
drush pmu ai_chatbot ai_assistant_api ai_provider_amazeeio ai key -y
drush config:delete ai_assistant_api.ai_assistant.ai_timeout_test
drush config:delete block.block.ai_timeout_test
drush config:delete key.key.ai_timeout_test_amazee
drush cr
```

## Caveats

- The amazee.ai API key is stored in **plaintext config** via the `key`
  module's config provider. Fine for a disposable test environment, not
  appropriate for production.
- Anonymous users cannot interact with the block (auth/admin roles only).
