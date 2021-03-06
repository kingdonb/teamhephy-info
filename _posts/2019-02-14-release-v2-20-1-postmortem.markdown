---
permalink: /blog/posts/announcements/release-v2-20-1-postmortem.html
layout: post
title:  "Release: Hephy Workflow v2.20.1 - Postmortem"
date:   2019-02-14
category: "announcements"
tags: [releases, 2019]
---

Hephy Workflow v2.20.1 (codename: "Semver and the Point Release") is out!

Users all should be warned that there have been some issues reported since it was released on Thursday, February 14, and some known bugs may now affect users of the current and previous release. Users please read all of this information carefully, especially if you have a Hephy Workflow installation in production.

GKE users take special note of this upgrade, as we have accidentally broken
some compatibility with the latest GKE releases, due to their lack of adherence
with SemVer standards. This break was an oversight on our part; we have not
made an emergency patch as the issue is quite complex and will take time to
communicate with upstreams, and ascertain precisely how to correctly resolve
without causing harm.

Another special warning for Hephy Workflow v2.20.0 users: platform admins should
remember to thoroughly understand backups and retention schedules, and always
test your platform database backups before an upgrade. Test this upgrade in
your staging environment first, if you have one, and as always recommended
according to best practices advice, regularly test your backups in any case.

Hephy users should run `helm repo update` first, to begin to safely mitigate the issue.

The buggy chart is no longer installable from `https://charts.teamhephy.com`,
our canonical charts repo, or ChartMuseum service. This is generally the first
step to receive updates from your chart repos, but in this case we have
actually overwritten a bad chart so that nobody will ever install it again.

#### Description of the Bug

If you have v2.20.0, look for the error in the lifecycle.preStop of your chart's
`workflow/charts/database/templates/database-deployment.yaml`:

```
lifecycle:
  preStop:
    exec:
      command:
        - su-exec
        - postgres
        - do_backup
```

Above, find the corrected version. If your template says `gosu` instead of
`su-exec` here, then you received the bad version of the chart. There is no
`gosu` binary in the postgres component image that is used when you install
`workflow-v2.20.0`, or its dependency `database-v2.7.1`.

The result can be a failure to create the critical *base backup* upon clean
termination of your image. The lifecycle `preStop` is supposed to guarantee
that even if your database does not run a full 4 hours, to normally execute a
scheduled base-backup, that this critical first base backup is created.

If your database has run for more than 4 hours, then the scheduled backup will
run. There is no `gosu` or `su-exec` with the scheduled backup, and this issue
should no longer affect you. We hope you will appreciate that we are reporting
this issue in the spirit of transparency, and an abundance of caution, but do
not be alarmed. Your cluster is probably not at risk, as I will explain here.

#### v2.20.0 PostgreSQL Database Total Loss Scenario - Postmortem

A dangerous issue which could corrupt some backups, was identified and silently
patched in the chart for v2.20.0 to prevent any preventable disasters, as there
were changes required in the chart related to
[feat(postgres): modify the docker base image as postgres:11-alpine #5](https://github.com/teamhephy/postgres/pull/5/files#diff-7927d6e9c88c978749d8c56320e281dd)
that got merged and released with v2.20.0, but unfortunately all changes were
not correctly reflected in the updated charts for [the postgres database](https://github.com/teamhephy/postgres).

Workflow installations configured with an off-cluster or "external" database
will not experience any issue related to this chart error. Clusters which have
been online for more than 4 hours also would not be negatively impacted.

We noticed the issue while we were beta-testing v2.20.1. A freshly installed
v2.20.0 with the default Minio object storage is at the greatest risk, as it is
an in-memory store and as configured by default, will not survive a reboot.
This also seems like the most common deployment that might be affected, as
Minio in the Team Hephy release trunks has been upgraded less recently than
Database.

The buried lede is that users can upgrade workflow-v2.20.0 to workflow-v2.20.1
in minikube, and see the new Database upgrade scripts working (or, not working)
depending on "which version of `v2.20.0`" you have, and how long you waited
after cluster init to perform the upgrade.

Folks who are not upgrading clusters on Minikube, and have a short cycle, or
cluster lifespan, are not likely to notice this bug at all. (In fact, that's
how we found it! In pre-release testing for v2.20.1. This issue was never
reported by any users, and **may never even have affected any clusters in the
wild at all, even once**.)

The bad chart was unfortunately pushed to charts.teamhephy.com repo and
circulated before the issue was identified. Properly configured clusters with
[Persistent Object Storage](https://docs.teamhephy.info/installing-workflow/configuring-object-storage/)
would not be at risk of data loss after the first 4 hours. **(Doesn't sound so bad now, does it?)**

Still, database users must be aware that this release represents a breaking
change for Workflow clusters with on-cluster databases, as upgrading will
automatically upgrade any earlier backups. NB: Restoring your database to the
old version after that will become a drastically higher level of difficult.

We will follow up this Postmortem later with a deep-dive that explains how
`deis-database` Postgres "WAL" backups work, and how users can verify that they
are working before they are needed. Thanks for reading, remember to join us on
Slack, and I hope you enjoyed this month's episode of the Hephy Postmortem show!
