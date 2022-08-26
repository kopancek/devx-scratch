# 2022 maintenance log

DevX teammates hacking on maintenance. Maintenance tasks encompass large migrations that we're tackling bit by bit, updating libraries, and all other improvements that can't be attached to a particular domain.

To add an entry, just add an H2 header starting with the ISO 8601 format, a topic.
**This log should be in reverse chronological order.**

## 2022-08-26

@jhchabran I'm creating this log entry for tracking our maintenance efforts.

### Rebooting our efforts to migrate our logs

I'm closing https://github.com/sourcegraph/sourcegraph/issues/36382 as it still references GitStart and I want to have a nice issue that all of us can pick and have all of what we need to use it as a filler task as we discussed in our team sync.
I have created https://github.com/sourcegraph/sourcegraph/issues/40896 which provides a clear path on how to do a migration on your own, without any external help. 

### Actual migration PR

I have submitted https://github.com/sourcegraph/sourcegraph/pull/40894 that migrates `workerutil.dbworker` to `github.com/sourcegraph/log`.

