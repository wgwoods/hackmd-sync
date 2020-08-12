# Packager Workflow

Hello! This document is an attempt to document the basic Fedora / CentOS
packager workflow and identify all the systems involved so we can figure out how to monitor the health of the overall system.

## Overview, packager PoV

Here's a step-by-step summary of what the Packager Workflow looks like, 
from the packager's point of view. This roughly matches the tests run by
the [monitor-gating] script. 

For more info about packaging procedures, see:
* https://fedoraproject.org/wiki/Package_maintenance_guide
* https://fedoraproject.org/wiki/Package_update_HOWTO


Systems involved in each step are listed in **bold**.

Messages associated with each step may vary as follows:

* `$ENV`: which environment we're using (`stg`, `prod`)
* `$CI_ENV`: which CI environment we're using (`stg`, `prod`)

### Prerequisites
1. Get Fedora account, sign CLA, etc
2. Get edit rights to a package (hereinafter: `$PKG`)

### Update package .spec / source
1. Log in / authenticate with **FAS**
    * Action: `kinit ...`
2. Clone package repo from **dist-git**
    * Action: `fedpkg clone $PKG`
3. Update `$PKG.spec`
    * Action: _[edit `$PKG.spec`]_, `git commit ...`
    * Note: maintaining .spec files, frankly, sucks
        * There are helper scripts / automated tools etc for this
        * Out of scope for this project tho
4. Upload new sources to **lookaside**
    * Action: `fedpkg new-sources ...`
5. Push updated `.spec`/`sources` to **dist-git**
    * Action: `fedpkg push`
    * **TODO**: server-side commit hooks / acls?

### (Optional) Submit PR, wait for CI results, merge PR
1. Create PR in **pagure**
    * Action: _[use Web UI or API to submit PR]_
2. Wait for PR-triggered **ci-pipeline**
    * Message: `org.centos.$CI_ENV.ci.dist-git-pr.test.running`
    * Message: `org.centos.$CI_ENV.ci.dist-git-pr.test.error`
3. Check CI status flag in **pagure**
    * **TODO**: Action?
4. Merge PR in **pagure**
    * Action: _[use Web UI or API to merge PR]_

### Build package
1. Submit build to **koji**
    * Action: `fedpkg build ...`
    * (Note: `fedpkg` waits for build completion unless passed `--nowait`)
2. Wait for **koji** to announce build completion
    * Message: `org.fedoraproject.$ENV.buildsys.build.state.change`
3. Check build tags in **koji**
    * Action: `koji call listTags $NEVR` (or use web UI)
    * Tags: `updates-candidate`, `signing-pending`

### Make update
1. Create update in **bodhi**
    * Action: `bodhi updates new ...`
2. Check for new tags in **koji**
    * Tags: `testing-pending`
3. Wait for **bodhi** to notify **ci-pipeline**
    * Message: `org.fedoraproject.$ENV.bodhi.update.status.testing.koji-build-group.build.complete`

### Wait for CI tests and review test results
1. Wait for **ci-pipeline** to run
    * Message: `org.centos.$CI_ENV.ci.koji-build.test.running`
    * Message: `org.centos.$CI_ENV.ci.koji-build.test.error`
2. Wait for results in **resultsdb**
    * Message: `org.fedoraproject.$ENV.resultsdb.result.new`
3. Wait for **greenwave** decision to accept/block build
    * Message: `org.fedoraproject.$ENV.greenwave.decision.update`
4. Check that package was signed by **robosignatory**
    * Tags: NOT `signing-pending`

### Waive test failures to unblock build
1. Tell **bodhi** to waive test results
    * Action: `bodhi updates waive $UPDATEID ...`
2. Wait for **waiverdb** to announce waiver
    * Message: `org.fedoraproject.$ENV.waiverdb.waiver.new`
3. Wait for **greenwave** to unblock build
    * Message: `org.fedoraproject.$ENV.greenwave.decision.update`
4. Wait for **koji** to tag build
    * Tags expected: `f33` (or whatever the target was)

At last: update complete! Grab a beer. You've earned it.

---------------------------------------------------------------------

## Overview, system PoV

Here's a (_WORK-IN-PROGRESS!_) list of systems/services/apps/etc. that are involved in this process. 

**NOTE**: A "system" is kind of an abstract concept. Take "FAS", for example -
the actual "Fedora Account System" is made of multiple services running on
multiple hosts. What we're trying to do here is identify the actual *services*
and *hosts* that make up each system, and figure out how to monitor each one's 
health and performance.

**NOTE 2**: In some cases we run multiple instances of a system, so (for example) just referring to "Jenkins" isn't enough - that could be [CentOS Jenkins], [fedora-infra Jenkins], [ci-pipeline Jenkins] or some other Jenkins instance.

On to the systems, in no particular order:

### FAS
The Fedora Account System is responsible for user authentication and related 
tasks - sometimes called "AAA", for Authorisation, Authentication, and
Accounting. The current [Fedora Account System] is the second version, so
it's also known as FAS2. We're currently building a new AAA for Fedora &
CentOS - based on [FreeIPA], with a new frontend called [noggin], so we'll
be moving from FAS2 to the new Fedora AAA Someday Soon(tm). But here's
what we have now.

#### FAS2 components
* Web frontend: https://admin.fedoraproject.org/accounts/
* 2FA backend: [totp-cgi] on `fas01`, `fas02`, `fas03`
* **TODO**: the rest of the backend pieces?


### dist-git
`dist-git` is the system that holds all the package `.spec` files and the 
"lookaside cache" of each package's sources. It's currently hosted in a
[pagure] instance but we're planning to move[^gitlab-move] to [gitlab]
Someday Soon(tm).

#### pagure
The public-facing web UI (and API) for dist-git.

* Web frontend: https://src.fedoraproject.org/
* **TODO**: Hosts
* **TODO**: Required/backend services

#### lookaside
The dist-git cache of package sources - tarballs, patches, you name it.

* **TODO**: Hosts
* **TODO**: Required/backend services


### fedmsg
Fedora systems communicate via a shared message bus. **fedmsg** is the
current Fedora message bus; it's considered deprecated[^fedmsg] and slated
to be replaced by **Fedora Messaging** in the near future.

* Hosts: `hub.fedoraproject.org`, ...?
* **TODO**: Required/backend services

#### datanommer
[datanommer] is a service that sits on the message bus and crams all the
messages it sees into a PostgreSQL database so [datagrepper] can query it.

* **TODO**: Hosts
* **TODO**: Backend/required services

[datanommer]: https://github.com/fedora-infra/datanommer/

#### datagrepper
[datagrepper] is a JSON API endpoint for querying the stored messages. [monitor-gating] uses [datagrepper] to look for messages that indicate that builds/tests/etc are complete.

* Web frontend: https://apps.fedoraproject.org/datagrepper/
* API endpoint: https://apps.fedoraproject.org/datagrepper/raw/
* **TODO**: Hosts
* **TODO**: Required/backend services

[datagrepper]: https://github.com/fedora-infra/datagrepper


### koji
This is the Fedora/CentOS build system. It handles building basically
every package, container, module, etc. that we publish.


* Web frontend: https://koji.fedoraproject.org/
* API endpoint: **TODO**
* Hosts: `koji.fedoraproject.org`, `kojipkgs.fedoraproject.org`, ...
* Backend: **postgresql**, ... (**TODO**: ntap? other things?)


### bodhi
Bodhi manages creating, testing, and publishing updates for Fedora.

* Web frontend: https://bodhi.fedoraproject.org/
* **TODO**: Hosts
* **TODO**: Required/backend services


### sigul
[sigul] is our automated GPG signing system. Its main purpose is to allow
us to sign packages, containers, etc. (collectively "artifacts") without
actually giving anyone access to the signing keys, which are isolated on
the signing server.

It's divided into 3 parts:

1. `sigul_server`: an isolated server which holds private keys,
2. `sigul_bridge`: bridge system that moves RPM data between Koji and the signing server
3. `sigul`: client code that talks to the bridge & server to sign data, manage keys, etc.

* **TODO**: Hosts
* **TODO**: Required/backend services

[sigul]: https://pagure.io/sigul

#### robosignatory
[robosignatory] is a [Fedora Messaging](#Fedora-Messaging) service that will
sign things (using [sigul]) in response to message requests.

* **TODO**: is this part of the current Packager Workflow?

[robosignatory]: https://pagure.io/robosignatory


### ci-pipeline / Jenkins
Pipelines! Pipelines! Everyone loves Jenkins Pipelines!
In the context of the Packager Workflow, [ci-pipeline] refers to the
Jenkins pipelines that get triggered in response to packager actions:

* [fedora-pr] handles PR-triggered CI
* [fedora-build] handles build-triggered CI tests

There's links to more info on [Fedora CI](#Fedora-CI) below.

* Hosts: https://jenkins-continuous-infra.apps.ci.centos.org/
* Backend: **jenkins**, baby. 

[ci-pipeline]: https://github.com/CentOS-PaaS-SIG/ci-pipeline
[fedora-pr]: https://jenkins-continuous-infra.apps.ci.centos.org/job/fedora-pr-new-trigger
[fedora-build]: https://jenkins-continuous-infra.apps.ci.centos.org/job/fedora-build-pipeline-trigger

### resultsdb
Hey hey, it's [resultsdb], the place where test results ~~go to die~~ are
stored for other people or services to use or examine! The [resultsdb core] 
handles the datastore and provides the API and the [resultsdb client] code
is used to interact with the API from the commandline.

[resultsdb core]: https://pagure.io/taskotron/resultsdb
[resultsdb]: https://fedoraproject.org/wiki/ResultsDB

* Web frontend:
  https://taskotron.fedoraproject.org/resultsdb/results
* API Client:
  [code](https://pagure.io/taskotron/resultsdb_api),
  [docs](https://resultsdb20.docs.apiary.io/)
* Hosts: `resultsdb-dev01.qa`, `resultsdb-stg01.qa`, `resultsdb01.qa`
  * (from [the resultsdb SOP](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/resultsdb.html), may not be up-to-date)
* Backend: **postgresql** [where?]

### greenwave
Greenwave is a **fedmsg** service that makes decisions about gating RPMs in
updates by consulting resultsdb and waiverdb. Fun!
For more info you might want to check out the [Greenwave SOP].

* Hosts: Fedora OpenShift, project `greenwave`
* Web frontend: https://greenwave-web-greenwave.app.os.fedoraproject.org
  * API endpoint: https://greenwave-web-greenwave.app.os.fedoraproject.org/api/v1.0
* Backend: Nope!
  * "Greenwave currently has no database (and we'd like to keep it that way)."

[Greenwave SOP]: https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/greenwave.html

### waiverdb
Waiverdb is a service for recording waivers, from humans, that correspond
with resultsdb. This is where you sign off on test failures to let gated
packages get pushed out. Check out the [waiverdb SOP] for more info!

* Web frontend: https://waiverdb-web-waiverdb.app.os.fedoraproject.org
* API endpoint: https://waiverdb-web-waiverdb.app.os.fedoraproject.org/api/v1.0/
* Hosts: Fedora OpenShift, project `waiverdb`
* Backend: **postgresql**

[waiverdb SOP]: https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/waiverdb.html

### Other systems to investigate

* [Koschei](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/koschei.html)


---------------------------------------------------------------------

## References / Further Reading

<!-- Links for [URL references] above. -->
[pagure]: https://pagure.io/pagure
[gitlab]: https://about.gitlab.com/
[monitor-gating]: https://pagure.io/fedora-ci/monitor-gating/
[totp-cgi]: https://github.com/mricon/totp-cgi
[ci-pipeline jenkins]: https://jenkins-continuous-infra.apps.ci.centos.org/
[fedora-infra jenkins]: https://jenkins-fedora-infra.apps.ci.centos.org/
[centos jenkins]: https://ci.centos.org/
[Fedora Account System]: https://admin.fedoraproject.org/accounts/
[AAA]: https://noggin-aaa.readthedocs.io/en/latest/
[noggin]: https://github.com/fedora-infra/noggin
[FreeIPA]: https://www.freeipa.org/

### Fedora CI

* [Fedora CI docs](https://docs.fedoraproject.org/en-US/ci/)
* [ci-pipeline](https://github.com/CentOS-PaaS-SIG/ci-pipeline)

### Fedora AAA

* https://github.com/fedora-infra/noggin
* https://github.com/fedora-infra/freeipa-fas

### fedmsg

* ZeroMQ: https://zeromq.org/
* Client library (Python): `fedmsg`
  ([src](https://github.com/fedora-infra/fedmsg/),
  [docs](https://fedmsg.readthedocs.io))
* Message topics: https://fedora-fedmsg.readthedocs.io
* Fedora CI message details: https://pagure.io/fedora-ci/messages
* Infra docs / SOPs:
  [intro](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/fedmsg-introduction.html),
  [gateway](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/fedmsg-gateway.html),
  [certs & signed messages](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/fedmsg-certs.html),
  [adding new message types](https://fedora-infra-docs.readthedocs.io/en/latest/sysadmin-guide/sops/fedmsg-new-message-type.html)


### Fedora Messaging

* AMQP: https://amqp.org/
* Hosts: `rabbitmq.fedoraproject.org`, ...
* Client library (Python): `fedora-messaging`
  ([src](https://github.com/fedora-infra/fedora-messaging),
  [docs](https://fedora-messaging.readthedocs.io))

<!-- footnotes -->

[^gitlab-move]: See the
  [blog post](https://communityblog.fedoraproject.org/making-a-git-forge-decision/),
  [gitlab issue](https://gitlab.com/gitlab-org/gitlab/-/issues/217350), and
  [endless fedora-devel thread](https://lists.fedoraproject.org/archives/list/devel@lists.fedoraproject.org/thread/QXJBN37CQRTVMKAYSS5PYVZXDPZZFZYN/#QXJBN37CQRTVMKAYSS5PYVZXDPZZFZYN)
  for details about the gitlab move.
  
[^fedmsg]: Yes, it's the classic software story: the current system is deprecated, but the replacement isn't finished, so everything is still running on the deprecated system.