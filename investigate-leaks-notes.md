The sections that follows contains some notes related to a couple of shutdown leaks investigated recently,
the examples are specific to WebIDL bindings for the WebExtensions APIs and refers also to classes not
yet landed on mozilla-central, but the strategies described are not tied to those specific cases and
should be useful for investigating leaks related to other areas too.

**WARNING NOTE**: this document describe my personal preference as a strategy to investigate leaks and it is
the strategy that I actually used to pin point the source of these particular leaks (and in the past for
other similar investigations), but make sure to refer to the official docs for more info about the topic.
In particular:

- for more info about gc and cc logs: https://firefox-source-docs.mozilla.org/performance/memory/gc_and_cc_logs.html
- for more info about the refcnt tracing: https://firefox-source-docs.mozilla.org/performance/memory/refcount_tracing_and_balancing.html

## Investigate leak introduced by a new WebIDL WebExtensions API binding using CC logs and refcnt tracing

I start by skipping all test tasks in test_ext_identity.html but one, aiming to get a run with the minimum
amount of "noise" which still reproduce the shutdown leak, and then I execute it with the environment variables
needed to make it produce the cc and gc edges log files:

```
$ MOZ_DISABLE_CONTENT_SANDBOX=t \
  MOZ_CC_LOG_DIRECTORY=/tmp/CC-LOGS \
  MOZ_CC_LOG_SHUTDOWN=1 \
  MOZ_CC_ALL_TRACES=shutdown \
  mach mochitest toolkit/components/extensions/test/mochitest/test_ext_identity.html --tag sw-webextensions
```

At the end of the test look at the BloatView for the process where the shutdown leak is detected:

```
 2:48.76 INFO leakcheck | Processing leak log file /tmp/tmpvv9atffr.mozrunner/runtests_leaks_tab_pid1926091.log
 2:48.76 INFO
 2:48.76 INFO == BloatView: ALL (cumulative) LEAK AND BLOAT STATISTICS, tab process 1926091
 2:48.76 INFO
 2:48.76 INFO      |<----------------Class--------------->|<-----Bytes------>|<----Objects---->|
 2:48.76 INFO      |                                      | Per-Inst   Leaked|   Total      Rem|
 2:48.76 INFO    0 |TOTAL                                 |       33     2064|  160153       26|
 2:48.77 INFO   83 |DOMEventTargetHelper                  |      128      256|      45        2|
 2:48.77 INFO  123 |ExtensionBrowser                      |      160      160|       1        1|
 2:48.77 INFO  124 |ExtensionIdentity                     |       72       72|       1        1|
 2:48.77 INFO  129 |ExtensionTest                         |       80       80|       1        1|
 2:48.77 INFO  186 |JS Object                             |        8        8|       1        1|
 2:48.77 INFO  238 |Mutex                                 |       72       72|     819        1|
 2:48.77 INFO  416 |RemoteServiceWorkerRegistrationImpl   |       48       48|       1        1|
 2:48.77 INFO  449 |ServiceWorkerGlobalScope              |      352      352|       3        1|
 2:48.77 INFO  452 |ServiceWorkerRegistration             |      208      208|       1        1|
 2:48.77 INFO  555 |WorkerEventTarget                     |      120      120|       2        1|
 2:48.77 INFO  556 |WorkerGlobalScope                     |      312      312|       3        1|
 2:48.77 INFO  557 |WorkerGlobalScopeBase                 |      232      232|       3        1|
 2:48.77 INFO  631 |mozilla::dom::ClientManager           |       48       48|       2        1|
 2:48.77 INFO  839 |nsStringBuffer                        |        8       96|  113822       12|
 2:48.77 INFO
 2:48.77 INFO nsTraceRefcnt::DumpStatistics: 912 entries
 2:48.77 INFO leakcheck: tab leaked 2 DOMEventTargetHelper
 2:48.77 INFO leakcheck: tab leaked 1 ExtensionBrowser
 2:48.77 INFO leakcheck: tab leaked 1 ExtensionIdentity
 2:48.77 INFO leakcheck: tab leaked 1 ExtensionTest
 2:48.77 INFO leakcheck: tab leaked 1 JS Object
 2:48.77 INFO leakcheck: tab leaked 1 Mutex
 2:48.77 INFO leakcheck: tab leaked 1 RemoteServiceWorkerRegistrationImpl
 2:48.77 INFO leakcheck: tab leaked 1 ServiceWorkerGlobalScope
 2:48.77 INFO leakcheck: tab leaked 1 ServiceWorkerRegistration
 2:48.77 INFO leakcheck: tab leaked 1 WorkerEventTarget
 2:48.77 INFO leakcheck: tab leaked 1 WorkerGlobalScope
 2:48.77 INFO leakcheck: tab leaked 1 WorkerGlobalScopeBase
 2:48.77 INFO leakcheck: tab leaked 1 mozilla::dom::ClientManager
 2:48.77 INFO leakcheck: tab leaked 12 nsStringBuffer
 2:48.77 UNEXPECTED-FAIL leakcheck: tab 2064 bytes leaked
```

From this logs I see that something is leaking a number of Extension API related classes,
along with some service worker related one.

Using [heapgraph's find_root.py script](https://github.com/amccreight/heapgraph/tree/master/cc)
I look for one of the classes leaked, and in particualr choosing one with just fewer entries
because it may make it simpler to pin point the issue.

Let's start by running `find_roots.py` to look for `ServiceWorkerGlobalScope` edges in the process `1926091`:

```
$ python2 ~/.../heapgraph/find_roots.py /tmp/CC-LOGS/cc-edges.1926091.log ServiceWorkerGlobalScope

Parsing /tmp/CC-LOGS/cc-edges.1926091.log. Done loading graph.

0x7f1b8a48d560 [ExtensionBrowser]
    --[mGlobal]--> 0x7f1bb3ad8b60 [ServiceWorkerGlobalScope ]

    Root 0x7f1b8a48d560 is a ref counted object with 1 unknown edge(s).
    known edges:
       0x7f1bb3ad8b60 [ServiceWorkerGlobalScope ] --[mExtensionBrowser]--> 0x7f1b8a48d560
       0x7f1b8a45b420 [ExtensionTest] --[mExtensionBrowser]--> 0x7f1b8a48d560
```

From this output it looks that
- one `ExtensionBrowser` instance is the reason for `ServiceworkerGlobalScope` being leaked
- and for that `ExtensionBrowser` instance there are two known edges (one related to 
  `ServiceWorkerGlobalScope` and one to `Extensiontest`) and an unknown edge... which is really suspicious.

That means that something is keeping a referece to that `ExtensionBrowser` instance but it is never releasing it.

By comparing `ExtensionTest` and `ExtensionIdentity` (which was also listed in the BloatView logs)
I see that:
- while `ExtensionTest` has a known edge to `mExtensionBrowser`
- `ExtensionIdentity` doesn't 

and that does also look quite suspicious and very likely the source of that unknown edge:

```
$ python2 ~/mozlab/cache/tools-mc/heapgraph/find_roots.py /tmp/CC-LOGS/cc-edges.1926091.log ExtensionIdentity
Parsing /tmp/CC-LOGS/cc-edges.1926091.log. Done loading graph.

0x7f1b8a48d560 [ExtensionBrowser]
    --[mExtensionIdentity]--> 0x7f1b8a45b470 [ExtensionIdentity]

    Root 0x7f1b8a48d560 is a ref counted object with 1 unknown edge(s).
    known edges:
       0x7f1bb3ad8b60 [ServiceWorkerGlobalScope ] --[mExtensionBrowser]--> 0x7f1b8a48d560
       0x7f1b8a45b420 [ExtensionTest] --[mExtensionBrowser]--> 0x7f1b8a48d560


$ python2 ~/mozlab/cache/tools-mc/heapgraph/find_roots.py /tmp/CC-LOGS/cc-edges.1926091.log ExtensionTest
Parsing /tmp/CC-LOGS/cc-edges.1926091.log. Done loading graph.

0x7f1b8a48d560 [ExtensionBrowser]
    --[mExtensionTest]--> 0x7f1b8a45b420 [ExtensionTest]

    Root 0x7f1b8a48d560 is a ref counted object with 1 unknown edge(s).
    known edges:
       0x7f1bb3ad8b60 [ServiceWorkerGlobalScope ] --[mExtensionBrowser]--> 0x7f1b8a48d560
       0x7f1b8a45b420 [ExtensionTest] --[mExtensionBrowser]--> 0x7f1b8a48d560
```

At this point it may be interesting to confirm if `ExtensionIdentity` is getting a reference to
`ExtensionBrowser` and never releasing it.

One way to achieve that is to rerun the test with the refcnt tracing logs for the class I'm interested
in and then review the output hoping to spot where the unknown edge is coming from:

```
XPCOM_MEM_LOG_CLASSES=ExtensionBrowser XPCOM_MEM_REFCNT_LOG=1 mach ...
```

and then review the `AddRef` and `Release` stacktrace log for the `ExtensionBrowser` class, looking for
entries that have `ExtensionIdentity` in the call stack for the `AddRef` and `Release` stacktraces.

Reviewing those logs confirms me that `ExtensionIdentity` does acquire a reference to `ExtensionBrowser`
but it is not releasing it (which is expected to happen as part of garbage and cycle collection, 
as it happens for ExtensionTest which can be confirmed also by reviewing the refcnt tracing logs related to it).

Reviewing `ExtensionIdentity.h/.cpp` will confirm that it was definitely the case, in particular
`ExtensionIdentity.cpp` was missing `mExtensionBrowser` as part of the
`NS_IMPL_CYCLE_COLLECTION_WRAPPERCACHE` macro.

## Investigate leak introduced by Extension API request forwarding using CC logs and rr

After fixing the leak described above, re-running the same test again does identify other leaks
(which were already there but hiding behind the other one), this time I choose to also record it
with [rr](https://github.com/rr-debugger/rr) to be able to replay the exact same execution if I need to:

```
$ MOZ_DISABLE_CONTENT_SANDBOX=t \
  MOZ_CC_LOG_DIRECTORY=/tmp/CC-LOGS \
  MOZ_CC_LOG_SHUTDOWN=1 \
  MOZ_CC_ALL_TRACES=shutdown \
  mach mochitest toolkit/components/extensions/test/mochitest/test_ext_identity.html \
    --tag sw-webextensions --debugger rr
 ...

 2:34.62 INFO leakcheck | Processing leak log file /tmp/tmps4ujjive.mozrunner/runtests_leaks_tab_pid2001995.log
 2:34.62 INFO
 2:34.62 INFO == BloatView: ALL (cumulative) LEAK AND BLOAT STATISTICS, tab process 2001995
 2:34.62 INFO
 2:34.62 INFO      |<----------------Class--------------->|<-----Bytes------>|<----Objects---->|
 2:34.62 INFO      |                                      | Per-Inst   Leaked|   Total      Rem|
 2:34.62 INFO    0 |TOTAL                                 |       28    24304|  196143     2264|
 2:34.62 INFO   24 |BackstagePass                         |      144      144|       1        1|
 2:34.62 INFO  186 |JS Object                             |        8      240|      30       30|
 2:34.62 INFO  238 |Mutex                                 |       72      216|     816        3|
 2:34.62 INFO  377 |PollableEvent                         |       32       32|       1        1|
 2:34.62 INFO  382 |Preferences                           |       88       88|       1        1|
 2:34.62 INFO  390 |Promise                               |       48       48|     242        1|
 2:34.62 INFO  461 |SharedPrefMap                         |       56       56|       1        1|
 2:34.62 INFO  498 |TelemetryImpl                         |      280      280|       1        1|
 2:34.62 INFO  576 |XPCNativeInterface                    |       56     1064|      41       19|
 2:34.62 INFO  577 |XPCNativeMember                       |       16      304|    1796       19|
 2:34.62 INFO  578 |XPCNativeSet                          |       32      480|      49       15|
 2:34.62 INFO  580 |XPCWrappedNative                      |       96     1440|     145       15|
 2:34.62 INFO  581 |XPCWrappedNativeProto                 |       40      240|      12        6|
 2:34.62 INFO  583 |XPCWrappedNativeTearOff               |       32      704|     176       22|
 2:34.63 INFO  627 |mozJSSubScriptLoader                  |       24       24|       1        1|
 2:34.63 INFO  662 |nsCategoryObserver                    |      104      104|       3        1|
 2:34.63 INFO  764 |nsIOService                           |      488      488|       1        1|
 2:34.63 INFO  773 |nsJSPrincipals                        |       24       24|     164        1|
 2:34.63 INFO  799 |nsObserverService                     |       80       80|       1        1|
 2:34.63 INFO  806 |nsPrefBranch                          |      112      224|       4        2|
 2:34.63 INFO  823 |nsScriptSecurityManager               |       64       64|       1        1|
 2:34.63 INFO  831 |nsSocketTransportService              |      376      376|       1        1|
 2:34.63 INFO  837 |nsStringBuffer                        |        8    16832|  150253     2104|
 2:34.63 INFO  888 |nsWeakReference                       |       40      320|     141        8|
 2:34.63 INFO  895 |nsXPCComponents                       |       88       88|       1        1|
 2:34.63 INFO  896 |nsXPCComponents_Classes               |       40       40|       1        1|
 2:34.63 INFO  897 |nsXPCComponents_Constructor           |       40       40|       1        1|
 2:34.63 INFO  899 |nsXPCComponents_Interfaces            |       40       40|       1        1|
 2:34.63 INFO  900 |nsXPCComponents_Results               |       40       40|       1        1|
 2:34.63 INFO  901 |nsXPCComponents_Utils                 |       40       40|       1        1|
 2:34.63 INFO  902 |nsXPCComponents_utils_Sandbox         |       32       32|       1        1|
 2:34.63 INFO  903 |nsXPCWrappedJS                        |      112      112|      77        1|
 2:34.63 INFO
 2:34.63 INFO nsTraceRefcnt::DumpStatistics: 910 entries
 2:34.63 INFO leakcheck: tab leaked 1 BackstagePass
 2:34.63 INFO leakcheck: tab leaked 30 JS Object
 2:34.63 INFO leakcheck: tab leaked 3 Mutex
 2:34.63 INFO leakcheck: tab leaked 1 PollableEvent
 2:34.63 INFO leakcheck: tab leaked 1 Preferences
 2:34.63 INFO leakcheck: tab leaked 1 Promise
 2:34.63 INFO leakcheck: tab leaked 1 SharedPrefMap
 2:34.63 INFO leakcheck: tab leaked 1 TelemetryImpl
 2:34.63 INFO leakcheck: tab leaked 19 XPCNativeInterface
 2:34.63 INFO leakcheck: tab leaked 19 XPCNativeMember
 2:34.63 INFO leakcheck: tab leaked 15 XPCNativeSet
 2:34.63 INFO leakcheck: tab leaked 15 XPCWrappedNative
 2:34.63 INFO leakcheck: tab leaked 6 XPCWrappedNativeProto
 2:34.63 INFO leakcheck: tab leaked 22 XPCWrappedNativeTearOff
 2:34.63 INFO leakcheck: tab leaked 1 mozJSSubScriptLoader
 2:34.63 INFO leakcheck: tab leaked 1 nsCategoryObserver
 2:34.63 INFO leakcheck: tab leaked 1 nsIOService
 2:34.63 INFO leakcheck: tab leaked 1 nsJSPrincipals
 2:34.63 INFO leakcheck: tab leaked 1 nsObserverService
 2:34.63 INFO leakcheck: tab leaked 2 nsPrefBranch
 2:34.63 INFO leakcheck: tab leaked 1 nsScriptSecurityManager
 2:34.63 INFO leakcheck: tab leaked 1 nsSocketTransportService
 2:34.63 INFO leakcheck: tab leaked 2104 nsStringBuffer
 2:34.63 INFO leakcheck: tab leaked 8 nsWeakReference
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_Classes
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_Constructor
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_Interfaces
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_Results
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_Utils
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCComponents_utils_Sandbox
 2:34.63 INFO leakcheck: tab leaked 1 nsXPCWrappedJS
 2:34.63 UNEXPECTED-FAIL leakcheck: tab 24304 bytes leaked
```

At a first glance this looks harder,those classes are much common ones and
the class names on their own are not that helpful to restrict the scope of the investigation,
some more clues are needed to be able to pin point what is triggering it.

Running heapgraph's `find_roots.py` on one class with fewer leaked instances is a
reasonable way to start, I choose to pick `Promise` because the identity API test case
is calling an async API method which returns a promise, and that looks something that could
be a more directly source of the leak (and everything else being leaked as a
side effect of leaking the Promise), and so let's look for `Promise`
class edges in the cc logs for the process `2001995`:

```
$ python2 ~/mozlab/cache/tools-mc/heapgraph/find_roots.py /tmp/CC-LOGS/cc-edges.2001995-1.log Promise
Parsing /tmp/CC-LOGS/cc-edges.2001995-1.log. Done loading graph.

0x7f09e2388d30 [Promise]

    Root 0x7f09e2388d30 is a ref counted object with 1 unknown edge(s).
```

That looks suspicious, the `Promise` instance at address `0x7f09e2388d30` does have 1 unknown edge.

`Promise` is a pretty common class, looking into it using the refcnt tracing logs will be likely producing
much more logs than I would like to read, but I recorded the same run with [rr](https://github.com/rr-debugger/rr)
and so I can instead try to pin pointing it by replaying the recorded session and adding some breakpoints
for the `Promise` instance at that address:

```
$ rr replay -p 2001995
...
(rr) b Promise::Promise if this == 0x7f09e2388d30
Function "Promise::Promise" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (Promise::Promise if this == 0x7f09e2388d30) pending.
(rr) continue
```

After a little bit `rr` hits the breakpoint set on the Promise constructor for the instance with an unknown edge:

```
Thread 1 hit Breakpoint 1, mozilla::dom::Promise::Promise (this=0x7f09e2388d30, aGlobal=0x7f09e9929790) at /home/rpl/mozlab/m-c/dom/promise/Promise.cpp:79
79          : mGlobal(aGlobal), mPromiseObj(nullptr) {
(rr) bt
#0  mozilla::dom::Promise::Promise(nsIGlobalObject*) (this=0x7f09e2388d30, aGlobal=0x7f09e9929790)
    at /home/rpl/mozlab/m-c/dom/promise/Promise.cpp:79
#1  0x00007f09fa0a5487 in mozilla::dom::Promise::CreateFromExisting(nsIGlobalObject*, JS::Handle<JSObject*>, mozilla::dom::Promise::PropagateUserInteraction)
    (aGlobal=0x7f09e9929790, aPromiseObj=..., aPropagateUserInteraction=mozilla::dom::Promise::eDontPropagateUserInteraction)
    at /home/rpl/mozlab/m-c/dom/promise/Promise.cpp:482
#2  0x00007f09fa0a5354 in mozilla::dom::Promise::Resolve(nsIGlobalObject*, JSContext*, JS::Handle<JS::Value>, mozilla::ErrorResult&, mozilla::dom::Promise::PropagateUserInteraction) (aGlobal=0x7f09e9929790, aCx=
    0x7f09e9224000, aValue=..., aRv=..., aPropagateUserInteraction=mozilla::dom::Promise::eDontPropagateUserInteraction)
    at /home/rpl/mozlab/m-c/dom/promise/Promise.cpp:124
#3  0x00007f09fe6b1b47 in mozilla::extensions::RequestWorkerRunnable::ProcessHandlerResult(JSContext*, JS::MutableHandle<JS::Value>) (this=0x7f09de847240, aCx=0x7f09e9224000, aRetval=...)
    at /home/rpl/mozlab/m-c/toolkit/components/extensions/webidl-api/ExtensionAPIRequestForwarder.cpp:559
#4  0x00007f09fe6b12dd in mozilla::extensions::RequestWorkerRunnable::HandleAPIRequest(JSContext*, JS::MutableHandle<JS::Value>) (this=0x7f09de847240, aCx=0x7f09e9224000, aRetval=...)
    at /home/rpl/mozlab/m-c/toolkit/components/extensions/webidl-api/ExtensionAPIRequestForwarder.cpp:523
...
```

This looks promising, the promise is definitely something related to a Extension API async method call.

Let's dig further by adding two more breakpoints to `Promise::AddRef` and `Promise::Release` conditioned on the specific `Promise` instance:

```
(rr) b Promise::AddRef if this == 0x7f09e2388d30
Breakpoint 2 at 0x7f09f3e2ded8: file /home/rpl/mozlab/m-c/obj-x86_64-pc-linux-gnu/dist/include/mozilla/dom/Promise.h, line 53.
(rr) b Promise::Release if this == 0x7f09e2388d30
Breakpoint 3 at 0x7f09f3e2b788: file /home/rpl/mozlab/m-c/obj-x86_64-pc-linux-gnu/dist/include/mozilla/dom/Promise.h, line 53.
(rr) continue
```

At this point `rr` hit the breakpoint on `Promise::AddRef` right after continuing the execution
it is clear that `Promise::Release` for that promise instance is never hit, even after letting `rr`
to continue until the end of the child process:

```
Thread 1 hit Breakpoint 2, mozilla::dom::Promise::AddRef (this=0x7f09e2388d30)
    at /home/rpl/mozlab/m-c/obj-x86_64-pc-linux-gnu/dist/include/mozilla/dom/Promise.h:53
53        NS_INLINE_DECL_CYCLE_COLLECTING_NATIVE_REFCOUNTING(Promise)
(rr) bt
#0  mozilla::dom::Promise::AddRef() (this=0x7f09e2388d30)
    at /home/rpl/mozlab/m-c/obj-x86_64-pc-linux-gnu/dist/include/mozilla/dom/Promise.h:53
...
#4  0x00007f09fa0a5494 in mozilla::dom::Promise::CreateFromExisting(nsIGlobalObject*, JS::Handle<JSObject*>, mozilla::dom::Promise::PropagateUserInteraction)
    (aGlobal=0x7f09e9929790, aPromiseObj=..., aPropagateUserInteraction=mozilla::dom::Promise::eDontPropagateUserInteraction)
    at /home/rpl/mozlab/m-c/dom/promise/Promise.cpp:482
(rr) continue
....
Thread 1 received signal SIGKILL, Killed.
0x0000000070000002 in syscall_traced ()
(rr)
```

This looks definitely the reason for the leak, looking to the code that is creating
that `Promise` from `mozilla::extensions::RequestWorkerRunnable::ProcessHandlerResult`
confirms it too:

the value returned by `dom::Promise::Resolve` is an `already_AddRefed<dom::Promise>`
and it should have been assigned to a `RefPtr<dom::Promise>`, which would recognize
that the refcnt for that already_AddRefed has been already incremented and it wouldn't
increment it again, and RefPtr would take care of releasing the promise and would not leak.

Fixing the issue as described above and rerunning the test will confirm that the shutdown
leak is gone and all other leaks were all side-effect of this single one.