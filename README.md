# PS5 Broadcast and Twitch Integration Investigation

## Scientific Research Summary — From Share/Broadcast Analysis to Runtime Capture Investigation

**Date:** 18 August 2026  
**Platform:** PlayStation 5, jailbroken/test environment

---

# 1. Research Objective

This project investigates the internal architecture of the PlayStation 5 Share/Broadcast subsystem, with two related research tracks:

1. Understanding the authorization and account-binding path that controls Broadcast.
2. Understanding the runtime capture/media path and how captured media moves into the streaming stack.

The investigation began as a general reverse-engineering study of PS5 Share/Broadcast functionality.

The initial questions were:

- Where is Broadcast implemented?
- How do Share, capture, transcoding, and streaming components interact?
- Which Sony system libraries expose relevant functionality?
- Where is service/account availability evaluated?
- Why does local fake sign-in change Broadcast UI availability?
- Why does Broadcast remain unavailable after fake sign-in?
- Where does Twitch authentication occur?
- How does the capture pipeline operate independently of the visible Broadcast UI?
- Can the existing PS5 capture/media pipeline eventually be consumed through another transport?

After the `np-fake-signin` experiment, the investigation developed a specific Twitch/provider hypothesis.

The current research questions are:

> What exact authorization and account-binding conditions stand between the local PS5 user state and `StartBroadcast`?

and:

> What is the actual media handoff path from the continuously active `SceAvCapture` pipeline into the streaming/media stack?

The Twitch/provider separation hypothesis remains unproven.

The media handoff path is also currently unresolved.

---

# 2. Initial Architectural Investigation

The first stage focused on identifying components involved in PS5 Share and Broadcast.

Analysis of Shell/VSH-related binaries revealed references to a Share wrapper layer:

```text
Sce.Vsh.ShareLibWrapper
```

and Broadcast/media-related concepts:

```text
SceShareVideoTapPoint
SceShareAudioTapPoint
```

These findings indicate that Broadcast is not simply one UI operation.

The system contains multiple layers responsible for capture, media handling, Share functionality, and streaming.

Additional components identified during the investigation included:

```text
CaptureContentUtil
CaptureGalleryUtil
SceVshShareVideoTranscoderWrapper
```

The initial architectural model was therefore:

```text
Shell/VSH
    ↓
Share/Broadcast services
    ↓
capture
    ↓
audio/video processing
    ↓
transcoding
    ↓
network/service layer
```

This is an architectural working model rather than a reconstructed call graph.

---

# 3. Native Share Function Mapping

The GarlicSaves Function Lookup database was used to map Sony Share-related APIs to their libraries.

Searching for:

```text
sceShare
```

revealed numerous Share-related functions.

Important examples include:

```text
sceShareInitialize
sceShareTerminate
sceShareFeaturePermit
sceShareFeatureProhibit
sceShareGetCurrentStatus
sceShareCaptureScreenshot
sceShareCaptureVideoClip
sceShareOpenMenuForContent
sceShareSetCaptureSource
sceSharePlayStartStreaming
sceSharePlayStopStreaming
sceSharePlayGetCurrentInfo
sceSharePlayGetCurrentConnectionInfo
```

Relevant libraries included:

```text
libSceShare.native.sprx
libSceSharePlay.sprx
libSceShareUtility.sprx
libSceShareFactoryUtil.sprx
```

This establishes that Sony separates several related concepts:

- capture;
- Share menus;
- Share Play;
- streaming;
- feature permissions;
- utility/UI operations.

The existence of generic Share APIs therefore does not by itself identify the exact authorization path used by live Broadcast.

---

# 4. Early Network and Media Findings

Static analysis of the relevant executable environment showed dependencies associated with:

```text
HTTP
SSL/TLS
networking
JSON
VideoOut
AudioIn
capture
```

This demonstrates that the relevant environment contains both media-processing and network/service functionality.

The presence of HTTP, TLS, and JSON support is consistent with Broadcast/provider operations being mediated through service APIs.

However, these dependencies alone do not identify the Twitch authentication mechanism.

---

# 5. Runtime Process Inventory

Runtime inspection established the following relevant processes in the test environment:

```text
SceShellUI
SceAvCapture
SceGameLiveStreaming
SceMediaCoreServer
webrtc_daemon.self
SceVideoCore2K
SceRemotePlay
```

The most important new runtime observation is:

```text
PID 60 — SceAvCapture
```

The investigation initially concentrated on:

```text
SceShellUI
```

because its runtime memory contained managed metadata associated with the visible Share/Broadcast interface.

The investigation has now expanded into:

```text
SceAvCapture
```

because it provides a concrete runtime target on the capture side.

Important distinction:

> The presence of multiple media/streaming processes does not yet prove which process consumes a particular capture structure.

That relationship still needs to be traced.

---

# 6. Service Availability Discovery

One of the earliest significant runtime discoveries was a managed metadata region containing:

```text
CheckServiceAvailabilityAllA
_sceNpAppInfoIntCheckServiceAvailabilityA
_sceNpAppInfoIntCheckAvailabilityA
```

alongside:

```text
SceNpTitleId
titleId
SceNpOnlineId
onlineId
reqId
userId
SceNpAppInfoLimitableFeature
feature
isAvailable
```

This establishes a relationship between the ShellUI managed layer and Sony's NP AppInfo service-availability functionality.

The conceptual structure is:

```text
ShellUI
    ↓
CheckServiceAvailability
    ↓
NP AppInfo feature
    ↓
availability result
```

The presence of:

```text
SceNpAppInfoLimitableFeature
```

and:

```text
isAvailable
```

shows that feature availability is represented explicitly in the software.

---

# 7. `liveStreaming` as a Feature

Runtime evidence later showed a Broadcast-related service availability check involving:

```text
feature = liveStreaming
```

This was an important discovery because it showed that the visible Broadcast UI state and the final ability to Broadcast cannot necessarily be treated as the same condition.

The working model became:

```text
UI availability
        ≠
complete Broadcast authorization
```

The exact sequence of checks remains unresolved.

---

# 8. ShellUI Broadcast Metadata

A search of the ShellUI executable region uncovered a concentrated metadata area containing:

```text
videoQuality
Youtube
isDisplayCameraEnabled
broadcastingAndScreenshareMode
FriendOnline
broadcastingAndScreenshareModeFromPlayStation
```

The same region also contained:

```text
sessionNameChanged
visibilityTypeChanged
otherUserStreamStarted
sessionListUpdateFailed
clientAllowed
clientConnecting
accountId
historyIds
sharePlayAction
requestAcs
```

These fields cover several independent concepts:

- streaming quality;
- provider identity;
- camera state;
- broadcasting mode;
- PlayStation broadcasting mode;
- session events;
- client state;
- account-related state;
- Share Play actions.

This strongly indicates that the ShellUI managed layer maintains a Broadcast/session model rather than merely presenting static UI elements.

---

# 9. Twitch Presence in ShellUI

Searching the mapped ShellUI image for:

```text
Twitch
```

produced multiple occurrences.

This establishes that Twitch is represented inside the ShellUI environment.

However, provider strings alone do not establish:

- where authentication occurs;
- where account binding occurs;
- where the access token is stored;
- whether the Twitch account is directly coupled to PSN identity.

Therefore:

> Twitch strings are useful for locating the subsystem, but they are not sufficient to reconstruct the authorization path.

---

# 10. Account and Client Metadata

Additional metadata was found for:

```text
clientAllowed
clientConnecting
accountId
```

These occurred alongside session-related metadata.

The important observation is that:

```text
accountId
```

in this context is a metadata/property name.

It should not be interpreted as an actual Twitch account identifier merely because the string exists in memory.

Therefore no claim is made that the discovered `accountId` occurrence represents a live Twitch identity.

---

# 11. `np-fake-signin` Experiment

The `np-fake-signin` project was used in the jailbroken/test environment to provide a local fake NP/PSN-like user context.

The important behavioral observation was:

```text
Before fake sign-in:
    Broadcast unavailable/disabled.

After fake sign-in:
    Broadcast became selectable.
```

This demonstrates that local user/sign-in state affects Broadcast UI eligibility.

It does not demonstrate that fake sign-in provides complete Broadcast authorization.

---

# 12. Remaining Broadcast Failure

After fake sign-in, selecting Broadcast produced:

```text
Can't broadcast this feature is unavailable for your account
```

This demonstrated that fake sign-in passes at least one earlier condition but does not satisfy the complete Broadcast authorization path.

The observed behavior can therefore be represented as:

```text
local user state
      ↓
Broadcast UI becomes available
      ↓
additional authorization/service checks
      ↓
Broadcast rejected
```

This was the point where the investigation moved from simply locating the Broadcast UI to reconstructing the authorization chain.

---

# 13. Provider/Service Log Evidence

Runtime/application logs from the Broadcast attempt showed service-availability activity involving:

```text
CheckServiceAvailability
```

and:

```text
liveStreaming
```

Provider-related operations included:

```text
connectAccount
partnerMetadata
youtube
twitch
```

Observed provider metadata indicated:

```text
youtube -> Permitted
twitch  -> Permitted
```

Despite this, Broadcast still failed with the account-unavailable error.

Therefore:

```text
provider listed/permitted
        ≠
Broadcast fully authorized
```

This makes the simple explanation:

```text
"Twitch is unsupported"
```

inconsistent with the observed evidence.

There must be another condition between provider availability and successful Broadcast startup.

The exact condition has not yet been identified.

---

# 14. Filesystem Investigation

The PS5 filesystem was examined for obvious local Broadcast/provider state.

Searches focused on:

```text
Twitch
YouTube
connectAccount
liveStreaming
partnerMetadata
```

The examined user-side filesystem did not expose an obvious plaintext Twitch OAuth/account credential file.

This does not prove that Twitch credentials or provider state are not stored locally.

It only establishes that the examined locations did not reveal a simple plaintext credential artifact.

Possible explanations remain:

- provider state is managed by another component;
- state is stored in protected form;
- credentials are obtained dynamically;
- relevant state exists outside the examined user directory;
- provider state is represented through a service rather than a simple file.

---

# 15. Discovery of the Dedicated Broadcast Module

The most important static artifact identified during the investigation was:

```text
Sce.Vsh.ShellUI.Broadcast.dll.sprx
```

This was a major improvement over relying on generic ShellUI strings.

The module contains explicit classes and methods related to:

- live streaming;
- provider permissions;
- Twitch authentication;
- account identity;
- Broadcast startup;
- stream configuration.

This provides a direct static-analysis target for reconstructing the Broadcast path.

---

# 16. Twitch-Specific Broadcast Service

The Broadcast module contains:

```text
GlsBroadcasterShareServiceTwitch
```

and Twitch-specific methods:

```text
GetGlsAccessTokenTwitch
SetGlsAccessTokenTwitch
GetAccessTokenTwitch
SetAccessTokenTwitch
c_initAccessTokenTwitch
```

This is direct evidence that Twitch is implemented as an actual Broadcast provider abstraction.

Twitch is therefore not merely a UI string.

There is dedicated provider state and provider-specific token handling.

---

# 17. Access and Refresh Token State

The Broadcast component contains generic authentication operations:

```text
GetAccessToken
SetAccessToken
SetRefreshToken
```

It also contains UI-facing token state:

```text
get_accessTokenFromUI
set_accessTokenFromUI
```

and token-update behavior:

```text
UpdateToken
```

This establishes that the Broadcast component has explicit token-related provider state.

Therefore the architecture cannot reasonably be reduced to:

```text
stream key only
```

The evidence supports at least three distinct categories:

```text
provider authentication
        +
provider/account state
        +
broadcast configuration
```

The exact relationship between these states remains unresolved.

---

# 18. Live-Streaming Permission APIs

The Broadcast component contains:

```text
IsPermitLiveStreamingAsync
IsPermitLiveStreamingIntAsync
```

Earlier runtime evidence had already identified:

```text
liveStreaming
```

as a service/feature involved in Broadcast availability.

The dedicated Broadcast component therefore contains explicit APIs representing live-streaming permission.

The exact implementation and return conditions still need to be traced.

---

# 19. Service-Provider Permission

The Broadcast module also contains:

```text
GetServiceProviderPermissionIntAsync
GetPermissionProviderListFromLiveStreaming
GetSupportedProviderListAsync
GetSupportedProviderListAsyncForBroadcaster
```

This indicates that provider permission is represented separately from generic live-streaming permission.

The current conceptual model is:

```text
Live-streaming availability
        ↓
Provider permission
        ↓
Provider/account state
        ↓
Broadcast
```

This is a working model.

The actual call order has not yet been reconstructed.

---

# 20. NP Account and Broadcasting Identity

The Broadcast module contains:

```text
GetNpAccountId
GetBroadcastingUserId
GetAccountId
GetOnlineId
GetTitleId
GetBroadcastTitleId
```

This demonstrates that the Broadcast layer explicitly handles PlayStation/NP identity while also maintaining broadcasting-related identity concepts.

This is relevant to the original hypothesis because it raises the possibility that several identity layers are represented internally.

However, the exact mapping between:

```text
NP account
broadcasting user
provider account
Twitch account
```

has not yet been reconstructed.

No claim is made that these identities can currently be decoupled.

---

# 21. Broadcast Start Operations

The Broadcast component contains:

```text
StartBroadcast
StartBroadcastFromGameTitle
StartIntAsync
```

These provide concrete endpoints for Broadcast startup.

The investigation can therefore focus on reconstructing the prerequisites leading to:

```text
StartBroadcast
```

instead of searching the entire ShellUI indiscriminately.

The main goal is to determine:

```text
Which checks must succeed before StartBroadcast?
```

---

# 22. Account-Binding and Permission States

The Broadcast module contains:

```text
AccountNotBind
NotPermittedByLiveStreamingMode
RemoveUnavailabeService
```

These states are highly relevant to the observed error:

```text
Can't broadcast this feature is unavailable for your account
```

However, their exact relationship to the visible error is not yet established.

In particular:

```text
AccountNotBind
```

demonstrates that an account-binding state exists internally.

It does not prove that:

```text
AccountNotBind
```

is the exact state currently causing the observed Broadcast failure.

That requires tracing the actual control flow.

---

# 23. Streaming-Key and Stream State

The Broadcast component contains:

```text
get_streaming_key
set_streaming_key
```

as well as:

```text
get_streamId
set_streamId
get_liveEventId
get_liveChatId
```

These demonstrate that streaming configuration is represented internally.

At the same time, the component contains explicit authentication state:

```text
GetAccessTokenTwitch
SetAccessTokenTwitch
SetRefreshToken
```

Therefore:

```text
streaming key
```

should not be treated as equivalent to:

```text
complete provider authentication
```

The current model is:

```text
provider authentication
        +
provider/account state
        +
broadcast configuration
```

---

# 24. Evolution of the Research Hypothesis

The investigation evolved through several stages.

## Stage 1 — General Share/Broadcast problem

```text
Where is Broadcast implemented?
```

## Stage 2 — Service availability

```text
Why is Broadcast considered unavailable?
```

## Stage 3 — Local user state

After fake sign-in:

```text
Why does Broadcast become selectable?
```

## Stage 4 — Provider state

After observing Twitch/YouTube provider metadata:

```text
Is provider availability separate from account authorization?
```

## Stage 5 — Twitch authentication

After discovering the Broadcast module:

```text
Does the Broadcast component maintain independent Twitch access-token state?
```

## Stage 6 — Runtime media path

After inspecting `SceAvCapture`:

```text
Does the PS5 maintain continuously active capture/history state
independently of pressing Record?
```

## Stage 7 — Current combined research question

```text
What exact authorization/account-binding conditions lead to StartBroadcast,
and what is the actual media handoff path from SceAvCapture into the
streaming/media stack?
```

---

# 25. Current Architecture Hypothesis

The current evidence supports two related but currently separate investigation paths.

## Broadcast authorization/control path

```text
                 LOCAL PS5 USER STATE
                          |
                          v
                    local NP context
                          |
                          v
                    Broadcast UI
                          |
                          v
              Service availability check
                          |
                          v
                 liveStreaming permission
                          |
                          v
                provider permission layer
                          |
              +-----------+-----------+
              |                       |
              v                       v
           YouTube                  Twitch
                                      |
                                      v
                              access/refresh token
                                      |
                                      v
                              account/binding state
                                      |
                                      v
                             broadcast configuration
                                      |
                                      v
                               StartBroadcast
```

This is a working authorization model.

It is not yet a reconstructed call graph.

---

## Media/capture path under investigation

```text
Game
  |
  v
SceAvCapture (PID 60)
  |
  v
continuously changing runtime state
  |
  v
????????????????????????????????
  |
  +------------------> SceGameLiveStreaming
  |
  +------------------> SceVideoCore2K / media processing
  |
  +------------------> SceMediaCoreServer
  |
  +------------------> downstream transport / WebRTC path
```

The second diagram is explicitly a research hypothesis.

The actual handoff from `SceAvCapture` to downstream media components has not been reconstructed.

---

# 26. Runtime Capture Investigation

The investigation has now moved into direct runtime observation of:

```text
SceAvCapture
```

The relevant process is:

```text
PID 60 — SceAvCapture
```

A runtime memory region was repeatedly observed at:

```text
0x00000201D8C000
```

Repeated reads of 16 bytes from this address showed the following pattern:

```text
00 00 30 00 xx xx xx xx xx xx xx xx xx xx xx xx
```

The first four bytes remained:

```text
00 00 30 00
```

in the observed samples.

The remaining fields changed continuously.

Representative observations included:

```text
00 00 30 00 44 7d 2e 00 5c 5b 0a 00 7b 04 00 00
00 00 30 00 c8 a0 0e 00 d0 7e 1a 00 7d 04 00 00
00 00 30 00 e4 f7 14 00 ec d5 20 00 7d 04 00 00
00 00 30 00 9c 78 1d 00 a4 56 29 00 7d 04 00 00
00 00 30 00 04 5b 2e 00 1c 39 0a 00 7d 04 00 00
00 00 30 00 bc aa 27 00 d4 88 03 00 84 04 00 00
00 00 30 00 38 70 2b 00 50 4e 07 00 87 04 00 00
00 00 30 00 50 ae 14 00 58 8c 20 00 88 04 00 00
00 00 30 00 38 a0 17 00 40 7e 23 00 89 04 00 00
00 00 30 00 68 9f 1d 00 70 7d 29 00 8e 04 00 00
```

Based on the observed little-endian representation, the structure can currently be described as:

```text
+00 : 0x00300000          stable in observations
+04 : changing uint32     rapidly changing
+08 : changing uint32     rapidly changing
+0C : changing uint32     generally increasing across samples
```

The important point is that these are **observations**, not decoded semantics.

The current evidence does not establish that:

```text
+04 = write pointer
+08 = read pointer
+0C = frame counter
```

or any other specific interpretation.

---

# 27. Important Interpretation of the Runtime Structure

The observed address:

```text
0x00000201D8C000
```

clearly contains live runtime state associated with an actively running component.

The structure changes even when no explicit Record operation is being performed.

This is consistent with the possibility that the PS5 maintains continuously active capture/history state.

However, the following have NOT been established:

```text
0x00000201D8C000 = video ring buffer
```

or:

```text
+04/+08 = buffer pointers
```

or:

```text
+0C = frame counter
```

or:

```text
the structure contains encoded video frames
```

The strongest defensible statement is:

> `SceAvCapture` exposes a continuously changing runtime structure at `0x00000201D8C000`, and the structure can be observed changing independently of a single Record operation.

This makes the address a useful target for further reverse engineering.

It does not yet identify the structure's exact type or ownership.

---

# 28. Record / Stop / Screenshot Correlation Experiment

Controlled runtime sampling was performed while exercising capture-related functions.

The tested behaviors included:

```text
normal gameplay / idle capture
short Record operations
longer recent-history capture
Record → Stop
Screenshot
```

The important observation is:

```text
the candidate structure continues changing during ordinary runtime
```

Therefore:

```text
changing bytes
```

alone cannot be interpreted as:

```text
Record button created these bytes
```

The correct next experiment is event-based correlation.

Conceptually:

```text
capture running
     ↓
Record pressed
     ↓
sample structure
     ↓
Stop pressed
     ↓
sample structure
     ↓
Screenshot pressed
     ↓
sample structure
```

The objective is to identify whether any field exhibits a deterministic transition associated with one of these operations.

So far, no such semantic mapping has been established.

---

# 29. Why the Capture Finding Matters

The earlier investigation established that the PS5 media environment contains multiple relevant processes:

```text
SceAvCapture
SceGameLiveStreaming
SceMediaCoreServer
SceVideoCore2K
webrtc_daemon.self
```

The new runtime finding provides a concrete location inside the capture side where continuously changing state can be observed.

This makes the next phase significantly more actionable than broad string searches alone.

The current high-value question is:

```text
What code reads or writes the state observed at
0x00000201D8C000,
and where does that state lead?
```

The goal is to identify the actual producer/consumer relationship.

---

# 30. Media Handoff Hypothesis

The long-term objective is not necessarily to reimplement the entire PS5 capture and encoding stack.

If the existing PS5 system already produces a reusable media representation, the preferable architecture would be:

```text
PS5 game
   ↓
existing SceAvCapture pipeline
   ↓
existing media/encoding path
   ↓
existing streaming/media component
   ↓
identified reusable interface
   ↓
our own consumer/transport
   ↓
custom stream endpoint
```

However, the current evidence does NOT prove that a direct consumer can be attached to:

```text
0x00000201D8C000
```

The address is currently a research target.

It is not yet a confirmed media-output interface.

Therefore the correct immediate objective is:

```text
identify the actual handoff
```

rather than:

```text
assume the observed memory is the media stream
```

---

# 31. Relationship Between Capture and Broadcast Research

The investigation currently contains two major paths:

```text
                    PS5
                     |
          +----------+----------+
          |                     |
          v                     v
   Broadcast control       Capture/media
          |                     |
          v                     v
  Authorization path      SceAvCapture
          |                     |
          v                     v
    StartBroadcast          unknown handoff
                                |
                                v
                       streaming/media stack
```

These paths may eventually converge.

However, the convergence point has not yet been identified.

It would therefore be incorrect to claim that:

```text
SceAvCapture
```

directly feeds:

```text
SceGameLiveStreaming
```

until code/runtime tracing demonstrates that relationship.

---

# 32. What Has Been Demonstrated

The following findings are directly supported by the investigation.

## Broadcast is a multi-component subsystem

Evidence includes Share wrappers, capture/tap points, transcoding, ShellUI components, and dedicated Broadcast services.

## Sony exposes native Share functionality

The Share function database contains numerous native Share APIs.

## Feature availability is explicitly modeled

ShellUI metadata contains NP AppInfo availability functions and feature state.

## `liveStreaming` is a specific service feature

Runtime evidence showed it being checked during the Broadcast flow.

## Fake sign-in changes Broadcast UI availability

Observed directly on the test PS5.

## Fake sign-in does not complete Broadcast authorization

Observed directly through the resulting account-unavailable error.

## Twitch is a recognized Broadcast provider

Observed in runtime/provider data and in the dedicated Broadcast component.

## Twitch has dedicated token APIs

Observed directly in:

```text
Sce.Vsh.ShellUI.Broadcast.dll.sprx
```

## Broadcast has explicit permission APIs

Observed directly in the Broadcast component.

## Broadcast handles NP identity

Observed through:

```text
GetNpAccountId
```

and related identity APIs.

## Broadcast has explicit start operations

Observed through:

```text
StartBroadcast
StartBroadcastFromGameTitle
StartIntAsync
```

## Broadcast has internal stream configuration state

Observed through:

```text
streaming_key
streamId
liveEventId
liveChatId
```

## `SceAvCapture` is actively running

Observed as:

```text
PID 60
```

in the test environment.

## A continuously changing runtime structure was identified

Repeated reads from:

```text
0x00000201D8C000
```

showed a stable first 4-byte value in the observed samples followed by changing runtime values.

## The structure changes independently of a single Record operation

The candidate structure continued to evolve during ordinary runtime.

This supports further investigation of a continuously active capture/history mechanism.

---

# 33. What Has NOT Been Demonstrated

The investigation has NOT demonstrated:

- that a Twitch access token can be inserted directly into the Broadcast component;
- that a stream key alone can start a Broadcast;
- that Sony-side live-streaming entitlement can be bypassed;
- that fake PS5 user state is sufficient for the Broadcast permission layer;
- that `AccountNotBind` is the exact state behind the visible error;
- that `GetNpAccountId` can be decoupled from Sony-side service authorization;
- that a genuine Twitch account alone is sufficient to make `StartBroadcast` succeed;
- that `0x00000201D8C000` is a video ring buffer;
- that the observed structure contains encoded video frames;
- that `+04` is a write pointer;
- that `+08` is a read pointer;
- that `+0C` is a frame counter;
- that the changing fields have a confirmed semantic meaning;
- that `SceGameLiveStreaming` directly consumes the observed structure;
- that `SceVideoCore2K` directly consumes the observed structure;
- that `SceMediaCoreServer` directly consumes the observed structure;
- that `webrtc_daemon.self` directly consumes the observed structure;
- that a direct memory-level media tap is possible;
- that the actual capture-to-streaming handoff has been reconstructed.

These remain open research questions.

---

# 34. Rejected or Weak Explanations

## "Broadcast is simply disabled"

Rejected as a complete explanation.

Fake sign-in changes the UI state.

Therefore there is at least one local user-state-dependent condition before the final Broadcast authorization stage.

---

## "Twitch is unavailable"

Not supported by current evidence.

Runtime provider information showed Twitch as permitted.

The remaining failure therefore requires another explanation.

---

## "There is no Twitch implementation"

Rejected.

The dedicated Broadcast module contains:

```text
GlsBroadcasterShareServiceTwitch
```

and Twitch token APIs.

---

## "The stream key is everything"

Not supported.

The Broadcast module contains both:

```text
streaming_key
```

and:

```text
access/refresh token
permission
account
```

state.

---

## "`accountId` in ShellUI is the Twitch account ID"

Not supported.

The observed occurrence is a metadata/property name.

---

## "The changing capture structure is proven to be raw video"

Not established.

The structure is demonstrably active, but its type, ownership, and semantics are unresolved.

---

## "The address is definitely the ring buffer"

Not established.

The correct current description is:

```text
candidate runtime capture-related structure
```

until code/data references establish its role.

---

# 35. Current Technical Targets

## Broadcast / authentication targets

```text
IsPermitLiveStreamingAsync
IsPermitLiveStreamingIntAsync

GetServiceProviderPermissionIntAsync
GetPermissionProviderListFromLiveStreaming
GetSupportedProviderListAsync
GetSupportedProviderListAsyncForBroadcaster

GetAccessTokenTwitch
SetAccessTokenTwitch
SetRefreshToken

GetNpAccountId
GetBroadcastingUserId
GetAccountId
GetOnlineId

StartBroadcast
StartBroadcastFromGameTitle
StartIntAsync
```

Important internal states:

```text
AccountNotBind
NotPermittedByLiveStreamingMode
RemoveUnavailabeService
```

Important configuration properties:

```text
streaming_key
streamId
liveEventId
liveChatId
accessTokenFromUI
```

---

## Capture / media targets

```text
SceAvCapture
PID 60

0x00000201D8C000

SceGameLiveStreaming
SceMediaCoreServer
SceVideoCore2K
webrtc_daemon.self
```

The highest-value runtime question is:

```text
Which code path produces, reads, writes, or consumes
the state observed at 0x00000201D8C000?
```

---

# 36. Next Research Phase

The next phase should combine static analysis and runtime tracing.

## Capture side

### 1. Correlate events

Perform controlled sampling around:

```text
Record
Stop
Screenshot
recent-history capture
```

and compare the exact field transitions.

### 2. Identify references

Search executable/data regions for references to:

```text
0x00000201D8C000
```

or for pointers that resolve to the observed region.

### 3. Determine field semantics

Establish whether:

```text
+04
+08
+0C
```

represent:

- pointers;
- offsets;
- sizes;
- timestamps;
- sequence numbers;
- counters;
- positions;
- indexes;
- or another structure type.

Do not assign semantics until supported by evidence.

### 4. Trace the producer

Determine which code writes the observed structure.

### 5. Trace consumers

Determine whether any of the following read the structure:

```text
SceGameLiveStreaming
SceVideoCore2K
SceMediaCoreServer
webrtc_daemon.self
```

### 6. Establish the media representation

Determine whether the data handed downstream is:

```text
raw frames
compressed frames
encoded packets
shared-memory buffers
DMA-backed surfaces
metadata/cursors
```

or another representation.

---

## Broadcast side

### 1. Trace permission evaluation

Follow:

```text
IsPermitLiveStreamingAsync
IsPermitLiveStreamingIntAsync
```

### 2. Locate failure states

Determine where:

```text
AccountNotBind
NotPermittedByLiveStreamingMode
```

are generated.

### 3. Trace provider permission

Follow:

```text
GetServiceProviderPermissionIntAsync
```

into provider/account state.

### 4. Trace Twitch token state

Determine how:

```text
GetAccessTokenTwitch
SetAccessTokenTwitch
SetRefreshToken
```

connect to provider authorization.

### 5. Trace identity

Determine the relationship between:

```text
GetNpAccountId
GetBroadcastingUserId
GetAccountId
GetOnlineId
```

### 6. Trace Broadcast startup

Reconstruct the prerequisites leading to:

```text
StartBroadcast
```

### 7. Correlate with the observed error

Determine which exact return state produces:

```text
Can't broadcast this feature is unavailable for your account
```

---

# 37. Priority Order

The most valuable next steps are currently:

```text
1. Identify references to 0x00000201D8C000
        ↓
2. Identify who writes the structure
        ↓
3. Identify who reads the structure
        ↓
4. Determine the semantics of +04/+08/+0C
        ↓
5. Reconstruct the capture/media handoff
```

In parallel:

```text
1. Trace IsPermitLiveStreaming*
        ↓
2. Trace provider permission
        ↓
3. Trace AccountNotBind / NotPermittedByLiveStreamingMode
        ↓
4. Trace Twitch token/account state
        ↓
5. Trace prerequisites to StartBroadcast
```

This approach should produce substantially more useful information than continued broad string enumeration.

---

# 38. Final Research Position

The project has progressed from the broad question:

> How does PS5 Share/Broadcast work?

to two narrower and testable questions.

First:

> What exact authorization and account-binding conditions stand between the local PS5 user state and `StartBroadcast`?

Second:

> What is the actual media handoff path from the continuously active `SceAvCapture` pipeline into the streaming/media stack?

The strongest current Broadcast-side evidence supports the following working model:

```text
Local PS5/NP user context
        ↓
Live-streaming availability
        ↓
Provider permission
        ↓
Twitch authentication/account state
        ↓
Broadcast configuration
        ↓
StartBroadcast
```

The strongest current media-side evidence supports:

```text
Game
  ↓
SceAvCapture
  ↓
continuously changing runtime capture-related state
  ↓
unknown handoff
  ↓
SceGameLiveStreaming / video/media processing
  ↓
transport
```

The `np-fake-signin` experiment demonstrates that local PS5 user state affects Broadcast UI availability.

The dedicated Broadcast module demonstrates that Twitch is represented through explicit provider-specific authentication/token APIs.

The runtime `SceAvCapture` investigation demonstrates that a continuously changing capture-related structure can be observed at:

```text
0x00000201D8C000
```

while `SceAvCapture` is running as:

```text
PID 60
```

However, the structure's exact semantics and its relationship to downstream media processing remain unresolved.

The project therefore has **two concrete reverse-engineering targets**:

```text
Broadcast authorization path
        +
Capture/media handoff path
```

The most important unresolved questions are now:

```text
What exact condition prevents StartBroadcast after fake sign-in?

and

What code path consumes the continuously changing state
observed inside SceAvCapture?
```

Those two questions should guide the next phase of the investigation.
