# Extension Activity Monitor 

**Mentor:** Luca Greco, ...

**Email:** lgreco@mozilla.com

## Project Description

Extensions do most of their work invisibly from users, and so  Extension activity is a complete mystery for most users, even advanced ones.

Providing more transparency could help increase reliability of abuse reporting and accountability for developers, as well as providing an
additional useful tool to aid investigating bugs in the extensions or in the Firefox WebExtensions internals.

## Skills Required

An applicant needs familiarity with:

* HTML/JavaScript and the Web API
* Revision Control Systems (e.g. git and/or mercurial)

and the following skills:

* Contributing to Open Source
* Think critically
* Good reading comprehension
* Do independent research
* Enjoy working with others
* Be productive with little supervision
* Able to work remotely

## Project Details

An `activityLog` API is available (in Firefox >= 70) to privileged extensions, the applicant goal is leveraging this API to
create a privileged extension which would explore various ways to present this information to the users.

In particular, this project goal is to expore the following areas:

- Browser tab (about:extensionmonitor, maybe) that shows all extensions active in all open tabs, extensions altering search, and for each installed extension
  show activity like network requests and data interactions between content scripts and background scripts.
- For the current tab, show extensions acting on it (maybe in the Site Information door hanger).
- Saving and loading activity logs from file

The applicant will also write a small API doc for the `activityLog` API, in [reStructuredText format](https://en.wikipedia.org/wiki/ReStructuredText)
as the [api docs available for the privileged `telemetry` API](https://searchfox.org/mozilla-central/rev/c52d5f8025b5c9b2b4487159419ac9012762c40c/toolkit/components/telemetry/docs/collection/webextension-api.rst)

Links to the `activityLog `API internals:
- API JSON schema: [activity_log.json](https://searchfox.org/mozilla-central/rev/c52d5f8025b5c9b2b4487159419ac9012762c40c/toolkit/components/extensions/schemas/activity_log.json)
- API test cases: [test_ext_activityLog.html](https://searchfox.org/mozilla-central/source/toolkit/components/extensions/test/mochitest/test_ext_activityLog.html)
- API implementation module: [ext-activityLog.js](https://searchfox.org/mozilla-central/rev/c52d5f8025b5c9b2b4487159419ac9012762c40c/toolkit/components/extensions/parent/ext-activityLog.js#17)
