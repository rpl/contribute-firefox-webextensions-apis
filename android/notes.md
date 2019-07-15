Contributing WebExtensions APIs to GeckoView
============================================

This page contains some personal notes and links related to:

- where to find additional docs and resources
- where the implementation (and related tests) lives
- how to contribute changes, how to test the applied changes and who to ask review and feedback from

## Help! I Have Another Question

You might not be alone! Look into these repo issues to see if someone else has asked your question.

- If they have, comment on the existing issue. If no one else has made an issue with your question, [file a new one][create-new-issue].

Do you want to suggest something to add/change/fix in this page?

Take a look at the existing issues and submitted pull requests on this repository, or [create a new issue][create-new-issue] or [pull request][create-new-pullrequest] if one doesn’t exist yet.
If you want to fix a typo or a broken link, just press the github edit icon action and create a pull request that fixes it. 

Happy Hacking!!!

[create-new-issue]: https://github.com/rpl/contribute-firefox-webextensions-apis/issues/new
[create-new-pullrequest]: https://help.github.com/articles/creating-a-pull-request/

## Additional docs and resouces

- [GeckoView docs][geckoview-docs]
- [Android bootcamp presentation @ Mozilla All Hands Whistler 2019][whistler2019-android-bootcamp-presentation]
- [WebExtensions/Contribution_Onramp wikipage][webext-contrib-wikipage]
- [WebExtensions/Hacking wikipage][webext-hacking-wikipage]

[geckoview-docs]: https://mozilla.github.io/geckoview/
[whistler2019-android-bootcamp-presentation]: https://docs.google.com/presentation/d/1MzU9q2wCwojC0kb1eVfma8hrQ-KayCRFFd_mV5Gx1F4
[webext-contrib-wikipage]: https://wiki.mozilla.org/WebExtensions/Contribution_Onramp
[webext-hacking-wikipage]: https://wiki.mozilla.org/WebExtensions/Hacking

## WebExtensions internals and APIs available in Android Builds

The WebExtensions internals implemented in `toolkit/` and `mobile/android` are shared between Fennec (Firefox for Android)
and GeckoView (used by various android-related projects, like Firefox Preview, Firefox Reality, Firefox for Fire TV and Echo Show).

Some WebExtensions APIs implemented in `toolkit/` are explicitly excluded in Android builds
(See section ["Exclude toolkit WebExtensions APIs from Android builds"](#exclude-toolkit-webextensions-apis-from-android-build) below).

The WebExtensions internals shared between Desktop and Android platforms lives in:

- `toolkit/components/extensions`: core WebExtensions internals, shared WebExtensions APIs implementations and schemas
- `toolkit/mozapps/extensions`: shared Add-ons Manager internals

The Android specific WebExtensions internals (shared between Fennec and GeckoView):

- `mobile/android/extensions`: WebExtensions APIs implementations and schemas

### Exclude toolkit WebExtensions APIs from Android builds

Some of the WebExtensions APIs implemented at toolkit level are excluded from Android builds
(usually because they wouldn't work in Android builds, and they would need an Android specific
implenmentation).

The WebExtensions API are excluded by surrounding them in a `#ifndef ANDROID` preprocessor macro
in the following two jar manifest files:

- toolkit/components/extensions/jar.mn
- toolkit/components/extensions/schemas/jar.mn

And conditionally registering the API module (based on AppConstants.MOZ_BUILD_APP), 
e.g. [See how the identity API is registered only on desktop builds here][conditional-registered-api-example]

[conditional-registered-api-example]: https://searchfox.org/mozilla-central/rev/282dea173ac29ed71cb0c1d37493c1d49d03ed79/toolkit/components/extensions/child/ext-toolkit.js#82-90

### Handling Differences between GeckoView and Fennec

There are some differences between GeckoView and Fennec that need to be taken into account
while working on the WebExtensions APIs shared between them:

- **Apps vs. Library**: 
  Fennec is a complete application, whereas GeckoView is an Android library, for this reason
  in GeckoView the requests originated from an extensions may need to be delegated to
  the GeckoView app (which should be able to allow or disallow the request based on the
  features the applications wants to support)

- **Multiprocess Architecture**:
  In Fennec both the extensions and the web pages runs in a single shared process,
  whereas in GeckoView the web pages runs in separate content processe, while the extensions
  runs in the main Gecko process.

In the WebExtensions APIs internals (and tests) it is possible to detect the kind of android build (Fennec or GeckoView)
using the `Services.androidBridge.isFennec` property.

Even if the extensions are currently running in the main Gecko process in both GeckoView and Fennec,
the implementation code should not assume that the extensions will always run in the main process.

### Other GeckoView internals related to extensions 

On the Gecko side:

- `mobile/android/modules/geckoview/GeckoViewWebExtension.jsm`: currently support the GeckoView nativeMessaging
  feature and allows the GeckoView app to register/unregister webextensions.
- `mobile/android/modules/geckoview/GeckoViewTab.jsm`: currently provide a GeckoView implementation of the BrowserApp
  global and Tab class (similarly to the internal abstractions currently provided by Fennec to manage tabs)
 
On the Java side:

- `org.mozilla.geckoview.GeckoRuntime`: provides the `registerWebExtension`/`unregisterWebExtension` methods
- `org.mozilla.geckoview.WebExtension`: provides the `WebExtension` class and the `WebExtension.MessageDelegate`
- `org.mozilla.geckoview.WebExtensionController`: provides the `WebExtensionController.TabDelegate`, which allows
  the GeckoView app to control some aspects of the WebExtensions tabs API (e.g. if the extension can successfully
  create a new tab using `tabs.create`)

### Other Fennec internals related to extensions 

- `mobile/android/chrome/content/aboutAddons.*`: Fennec about:addons page

## Test Suites

This section provides some notes related to the available test suites, what they cover and how to run them
(locally on an emulator and/or in a "try push").

### linting

JavaScript code should pass successfully eslint rules (and formatted using prettier):

```
## lint a specific set of files
$ ./mach lint [path/to/file.js ...]

## auto-fix linting issues on a specific set of files
$ ./mach lint --fix [path/to/file.js ...]

## prettier auto-format on current changes
$ ./mach prettier-format
```

Java code should pass the [android-checkstyle][android-checkstyle], [android-findbugs][android-findbugs] and [android-lint][android-lint] checks:

```
$ ./mach android checkstyle
...
$ ./mach android findbugs
...
% ./mach android lint
```

New public GeckoView APIs should pass the [apilint][android-apilint] checks:

```
$ ./mach android api-lint
....
```

[android-checkstyle]: https://developer.mozilla.org/en-US/docs/Mozilla/Android-specific_test_suites#Android-checkstyle
[android-findbugs]: https://developer.mozilla.org/en-US/docs/Mozilla/Android-specific_test_suites#android-findbugs
[android-lint]: https://developer.mozilla.org/en-US/docs/Mozilla/Android-specific_test_suites#android-lint 
[android-apilint]: https://mozilla.github.io/geckoview/contributor/geckoview-quick-start#updating-the-changelog-and-api-documentation 

### xpcshell tests

xpcshell tests are often used to unit test WebExtensions internals and APIs implemented at `toolkit/` level,
these tests run in a Gecko environment provided by the `xpcshell` binary, and so there is no GeckoView or Fennec
internals available to them.

These tests can be executed in an android emulator by running the following command:

```
./mach xpcshell-test /path/to/test/dir/or/file
```

The test harness will upload all the needed assets to the android emulator, execute the xpcshell tests on the device
and collect and print the test logs in the terminal shell where the mach command is running.

### xpcshell test troubleshooting

#### adb logcat

If a test fails or it is getting stuck and it is timing out, some additional logs (most of the times including 
the underlying error) are likely visible in adb logs, and so it may be useful to keep an `adb logcat` command
running in another terminal shell.

#### xpcshell segfaults before running any test case

If the xpcshell executable is segfaulting before it is actually executing any test case, it may be an issue
related to the emulator instance, and you may try to:

- update the emulator instance (using `mach android-emulator --force-update`)

- create a new "vanilla" emulator instance using the android sdk (from Android Studio, or from the command line
  using a command like `android create avd -k "system-images;android-29;google_apis;x86" -n EMULATOR_NAME`) and
  then starting the new emulator instance before executing the `mach xpcshell-test` command
  (`/path/to/Android/Sdk/emulator/emulator @EMULATOR_NAME`)

### mochitests

mochitests are tests running in a more realistic test environment:

- GeckoView (which is now the default target for mochitest running on Android builds): the mochitests
  test files runs in the test environment provided by [`org.mozilla.geckoview.test`][org-mozilla-geckoview-test]

- Fennec: the mochitests test files runs in a Fennec build instance

```
# Run mochitests on GeckoView

$ ./mach mochitest -f plain [--app org.mozilla.geckoview.test] path/to/mochitest/dir/or/file

# Run mochitest on Fennec

$ ./mach mochitest -f plain --app org.mozilla.fennec_USERNAME path/to/mochitest/dir/or/file
```

[org-mozilla-geckoview-test]: https://searchfox.org/mozilla-central/source/mobile/android/geckoview/src/androidTest/java/org/mozilla/geckoview/test 

### mochitest test troubleshooting

#### adb logcat

Similarly to the xpcshell test, more detailed logs related to a failure may be visible in adb logs,
and so it may be useful to keep an `adb logcat` command running in another terminal shell.

### junit

Both GeckoView and Fennec are also covered by junit tests. For GeckoView these kind of tests are also going to require
changes when a WebExtensions API is also paired with some new public GeckoView API (e.g. additions or changes to the
WebExtensions delegates).

```
$ mach geckoview-junit org.mozilla.geckoview.test.WebExtensionTest
```

### try push syntaxes

To run all the android tests on try:

```
./mach try fuzzy -q "android"
```
