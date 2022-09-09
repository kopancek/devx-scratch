# 2022 `cla-bot` log

DevX teammates hacking on `cla-bot`. To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-09-08

@bobheadxi

Set up alerting for clabot via a new Sentry project via `SENTRY_DSN` set in the repository secrets:

- [Issues](https://sentry.io/organizations/sourcegraph/issues/?project=6732169)
- [Alerts for #dev-experience-alerts](https://sentry.io/organizations/sourcegraph/alerts/rules/clabot/12167423/details/)

The main moving part of our CLA bot setup is the `sync` tool, which I've documented here: https://github.com/sourcegraph/clabot-config/tree/main/tools

General guidance of how the CLA bot works is linked to from here: https://github.com/sourcegraph/clabot-config

## 2022-06-21

@bobheadxi the new `cla-bot` automation has shipped, described in https://github.com/sourcegraph/sourcegraph/pull/37395

I ended up going with a fully automated approach, which *additively* syncs signatures from the Google form on a regular basis.
The code is pretty minimal, and also gives us some nice features like automatic sorting of usernames, fixes to common mistakes (adding a `@` for example), and so on.

https://sourcegraph.com/search?q=context:global+repo:%5Egithub%5C.com/sourcegraph/clabot-config%24%40main+file:.*%5C.go+

Along the way, had to fix the form which was set up incorrectly, leading to lots of entries that had incomplete data (namely missing a GitHub username) - see [thread in #legal](https://sourcegraph.slack.com/archives/C01GJFE26BX/p1655850127612279).

## 2022-04-25

@bobheadxi Proposal for `cla-bot` automation: https://github.com/sourcegraph/sourcegraph/issues/26992#issuecomment-1109113311

## 2022-04-19

@jhchabran

[David and Philip are merging the JetBrains PR](https://sourcegraph.slack.com/archives/C07KZF47K/p1650363090295009) in our monorepo and their PR cannot pass without the CLA-Bot approval. It's an egde case because those contributions are old and the `cla-bot` is triggered because we're importing the history of the repo they're importing the code from.
