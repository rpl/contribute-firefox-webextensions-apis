If you are interested in [Outreachy](https://www.outreachy.org/) and/or the project [“Firefox for Android WebExtensions API”](https://wiki.mozilla.org/Outreachy#Android_WebExtensions) and you want to know more (much, more),
**you have come to the right place!**

## Outreachy

If you want to apply for an Outreachy internship, you should take a look at these pages and ensure that you are eligible:

- https://www.outreachy.org/apply/eligibility/
- https://wiki.mozilla.org/Outreachy/Handbook

## Onboarding to the Codebase

Ok ok, you probably want to know something more about the project itself. This page includes information about what you should know, what you should read and try, and most importantly, where to find the code!

Well, there is a lot of code, don’t worry. :-)

Let’s start from the beginning!

### What is an extension? How I can create and run an extension on Firefox Desktop and Firefox for Android?

Working on WebExtensions APIs will make much more sense if you know how the developers and users are using what you are going to create.

Luckily, the awesome people from MDN web docs have created some great resources to help you to get started:

- https://developer.mozilla.org/Add-ons/WebExtensions
- https://github.com/mdn/webextensions-examples

How to run and debug an extension on Firefox Desktop:

- https://developer.mozilla.org/Add-ons/WebExtensions/Temporary_Installation_in_Firefox
- https://developer.mozilla.org/Add-ons/WebExtensions/Debugging
- https://developer.mozilla.org/Add-ons/WebExtensions/Getting_started_with_web-ext

How to run and debug an extension on Firefox for Android:

- https://developer.mozilla.org/Add-ons/WebExtensions/Developing_WebExtensions_for_Firefox_for_Android
- https://developer.mozilla.org/Add-ons/WebExtensions/Differences_between_desktop_and_Android

### How to get started on Firefox Desktop and Firefox for Android development:

Working on WebExtensions APIs means working on Firefox source code. This is exciting but it can also be scary! There is so much code inside a browser and in more than one programming language, but don’t be afraid. You don’t have to read all that code at once. You do need to learn how to read and understand it, how to search in the Firefox sources, and how to make and test your own changes.

Here are the MDN docs about how to clone sources and configure the development environment and build with artifacts (mixing your local changes to a prebuilt version of the Firefox sources, so that you don’t need to rebuild all the Firefox C++ and Java sources):

- https://developer.mozilla.org/docs/Mozilla/Developer_guide/Build_Instructions/Simple_Firefox_build
- https://developer.mozilla.org/docs/Mozilla/Developer_guide/Build_Instructions/Simple_Firefox_for_Android_build#I_want_to_work_on_the_front-end

Once you have been able to build Firefox successfully, you should try to run the Firefox for Android tests on the emulator:

- https://wiki.mozilla.org/Mobile/Fennec/Android/Testing#Running_tests_on_the_Android_emulator

Some wiki pages related to hacking on the WebExtensions internals (note: these can be outdated but still useful):

- https://wiki.mozilla.org/WebExtensions/Hacking#Code_layout
- https://wiki.mozilla.org/WebExtensions/Implementing_APIs_out-of-process

The best and more updated docs are usually the sources themselves, and searching for particular piece of code or identifier names in the Firefox sources is fast and simple with searchfox:

- http://searchfox.org/

Firefox and WebExtensions internals code written in JavaScript often uses some of new ES6/ES7 features and syntaxe. Here are some links to learn more about them:

- https://github.com/mbeaudru/modern-js-cheatsheet/blob/master/readme.md
- http://es6-features.org/

## Help! I Have Another Question

You might not be alone! Look into these repo issues to see if someone else has asked your question. If they have, comment on the existing issue. If no one else has made an issue with your question, [file a new one][create-new-issue].

Do you want to suggest something to add/change/fix in this page?

Take a look at the existing issues and submitted pull requests on this repository, or [create a new issue][create-new-issue] or [pull request][create-new-pullrequest] if one doesn’t exist yet.

Happy Hacking!!!

[create-new-issue]: https://github.com/rpl/contribute-firefox-webextensions-apis/issues/new
[create-new-pullrequest]: https://help.github.com/articles/creating-a-pull-request/
