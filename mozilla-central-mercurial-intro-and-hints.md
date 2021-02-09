## Mercurial-based mozilla-central workflow intro

### Other resources

Mike Conley did share some notes about his workflow in this blogpost:
- https://mikeconley.github.io/documents/How_mconley_uses_Mercurial_for_Mozilla_code

that is definitely a useful read.

### Intro

I usually start by double-checking where am I in the particular clone of the mozilla-central repo I'm entering:

```
cd mozilla-central
hg log -r .
```

Then if I'm starting to work on something I make sure to start with an updated mozilla-central tip:

```
hg update -r central
hg pull -u central
```

And then I create a new bookmark, if I'm the one picking the bookmark name (moz-phab patch will use its own convention by defaults) I try to follow a convention that more easily reming myself what the bookmark is about.

I often use the bug number and a date (because I usually create a new bookmark when I rebase the patch between two commits far from each other in the mozilla-central history):

```
hg bookmark bug-NNN--YYYYMMDD
```

> **tip:** _using a shell prompt that does also show you the current bookmark name is pretty useful to don't lose track of what you are working on (especially useful if someone is moving between different tasks over a days / weeks)_.

At some point the bookmark may include more than one commit that are meant to stay separated (and be reviewed and landed as a stack of patches), and we are going to get reviews on each of them separately.

The commits in the bookmark may be all related to the same bugzilla issue, or related to multiple bugzilla issues that depends on each other.

When updating one of the patches to handle the review comments, we want to:
- keep those commits separated
- any conflicts resolved

Also, from time to time we will have to rebase the entire stack of patches on top of a more recent mozilla-central tip over the time, e.g. because:

- we need to double-check that no merge conflicts would be triggered when we are going to land the patches
- or because there were other changes in mozilla-central that will conflicts or affect the patches in our bookmark

There are some mercurial commands that will be helpful to achieve that, but to make the process as less painful as possible we also need to be careful about how we split out patches and keep in mind how each of them relates to each other
(so that we can be prepared upfront as much as possible to what we'll need to do and in which order would be better to do it).

### Evaluate the clone status (hg status / hg diff / etc.)

I do double-check the status of the branch and what uncommitted changes I do have pretty often, e.g. to plan what to commit and which patch it should be part of, what shouldn't be committed at all (e.g. temporary debugging statements and other temporary changes not meant to be part of the actual patch in review).

- `hg status`
- `hg diff [path/to/a/mozilla-central/subdir/]`

When looking to the comming in the bookmark I'm currently working on, I'm only interested to see the commits in the current bookmark (and filter out any other commit from my other bookmark, as well as any other commit that I just got from mozilla-central):

```
hg log -r 'ancestors(.)&draft()'
```

This is so common that I can't leave without the following aliases (which help me typing less) defined in my hgrc config file:

```
[revsetalias]
pending = ancestors(.)&draft()
s($1) = sort($1, -rev)

[alias]
lg = log -r 'pending'
lgs = log -r 'pending' --stat
lgd = log -r 's(pending)'
lgp = log -pr 'pending'
lgpd = log -pr 's(pending)'
```

With this aliases defined I can focus on my current bookmark and type less:

```
hg lg    # just list the pending commit (no stats or raw patch details)
hg lgs   # list pending commit with stats
hg lgp   # list pending commit with raw patch

hg lgd   # list pending commit in reverse order (last commit listed first)
...
```

### hg commit --amend / hg amend

The easiest case is usually: I do have some pending changes that I want to fold into the last commit in this bookmark:

```
hg commit --amend
# or
hg amend
```

If I want to fold only part of the pending changes I add the `-i` flag (`-i` => `interactive mode`):

```
hg amend -i
# or
hg commit --amend -i
```

Mercurial will show me an text-based interactive UI, that will let me choose from the pending changes what I want to include and/or exclude.

The interactive mode may also be useful when creating new commits (for the same purpose, don't commit part of the pending changes).

### hg absorb

`hg absorb` is a recent addition to mercurial default plugins, and it does help to speed up the workflow when we need to fold a change into a commit that isn't the last one in the bookmark.

> **don't drink too much kool-aid**: `hg absorb` will work great if we try to use it on a set of patches where each patch do not touch the same lines changes in other patches from the same patch stack. The linting commit hook does not run when absorbing changes and so there are cases when the change would be absorbed just fine but the resulting patch may not lint successfully on its own.

`absorb` is a default plugin but it has to be enabled in the hgrc config:

```
[extensions]
absorb =
show =
strip =
share =
histedit =
rebase =
extdiff =
firefoxtree = ~/.mozbuild/version-control-tools/hgext/firefoxtree
clang-format = ~/.mozbuild/version-control-tools/hgext/clang-format
js-format = ~/.mozbuild/version-control-tools/hgext/js-format
push-to-try = ~/.mozbuild/version-control-tools/hgext/push-to-try
```

these are all the extension that I do have enabled locally (some of them have been added automatically by `./mach bootstrap`).

When you run `hg absorb` it will show you which lines would be absorbed into which patch (it may be more than one) and ask your confirmation before proceeding.

`hg absorb path/to/mozilla-central/subdir/or/file` may be useful in some cases, when we want to only absorb the pending changes from a particular fire or directory.

### hg histedit

Right after `absorb`, `hg histedit` is my second favorite way to work on multiple commits.

`hg histedit -d central` will show us a text-based interactive UI that can be used to:
- move the patches around (and be warned if there are conflicts risks when moving a patch above another one touching the same file around the same lines)
- rewrite commit messages
- fold a patch into another
- stop on a commit to amend it (and then rebase the rest of the patches on top of the amended commit)

I often prefer to use `histedit` instead of `hg absorb` when I know that I want to accumulate the pending changes into new commits before actually absorbing them into the other patches (e.g. because I'm still evaluating if the entire set of changes is going to workout as I think it should)

### hg export / hg import

When I can't absorb or reorder the patches without triggering more conflicts than I would like to solve on the fly while mercurial is rebasing the patches, I opt for a slightly more manual workflow based on `hg export` and `hg import` commands.

I have to resort to that often enough and so I do have some more aliases that help me to type less.

In my hgrc:
```
[alias]

ep = export -g -r 'pending' -o "$1/%n-%m.patch"
```

This alias allows me to export in a given directory all the pending commits from the bookmark I'm currently working on.

```
mkdir -p ../exported-patches/YYYYMMDD-bug-NNN/before-rebase
hg ep ../exported-patches/YYYYMMDD-bug-NNN/before-rebase
```

Each exported patch will be named as "NUMBER-COMMITMESSAGE.patch", which make it easy to immediately see:
```
ls ../exported-patches/YYYYMMDD-bug-NNN/before-rebase

01-Bug_NNN__text_from_commit_message.patch
02-Bug_NNN__text_from_commit_message.patch
03-Bug_NNN__text_from_commit_message.patch
...
```

The exported patches will still include the "Differential Revision" text in their commit description message and so when I re-import them I can push them to phabricator to update the related phabricator revision as usually:

```
# HG changeset patch
# User Luca Greco <lgreco@mozilla.com>
# Date 1610980887 0
#      Mon Jan 18 14:41:27 2021 +0000
# Node ID ...
# Parent  ...
Bug NNN - text from commit message.

...

Differential Revision: https://phabricator.services.mozilla.com/DNNNNN
```

After I have all patches exported, I can then move to the most recent mozilla-central tip and start re-importing the patches one by one, resolving conflicts (and running tests that I often try to add to each commit to more confortably rebase my bookmark, because the tests failure should let me know if I'm breaking my changes earlier then if I didn't have any test coverage at all):

```
# move back to the most recent mozilla-central commit I do have locally 
hg update -r central

# pull more recent mozilla-central commits into the local clone
hg pull -u central

# I create a new bookmark
hg bookmark bug-NNN--YYYYMMDD

# and then I start to re-import the patches using --partial
hg import --partial ../exported-patches/YYYYMMDD-bug-NN/before-rebase/01-...
```

If there are conflicts, `hg import --partial` would still create a commit for the imported patch (it could be even empty if the entire patch did have conflicts) I do:
- go through all the `*.rej` files and re-import those changes by adapting them to apply cleanly
- run the tests I think would help me catch mistakes I may be making by adapting the changes that conflicted (or if I need to apply more changes to deal with failure triggered by other unrelated changes in mozilla-central, that may not even trigger any conflicts but change some of the assumptions from the original patch)
- amend the imported commit
- remove all `*.rej` files I did already handle
- move to importing the next patch and continue until I did import all of them

### hg evolve

I don't use `hg evolve` that much, when `hg absorb` and `hg histedit` are not enough I usually opt for `hg export` and `hg import` to be a bit more in control of the steps needed to update my patches (and resolve conflicts if needed).

### selectively update pending patches on phabricator

When I do have more than one patch in a bookamrk, it is quite common that I may want to selectively update some of them on phabricator, `moz-phab submit` does support this use case, we just need to tell moz-phab which are the start and end revision that we want to submit on phabricator:

```
hg lg
... # look to the pending commit and take note of the start and end revision to push on phabricator

moz-phab submit REVID_START REVID_END -m "Brief comment about what I did change in this particular update"
```
