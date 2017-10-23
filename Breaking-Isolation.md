One of the core TAB/RM functions outlined in the TCG spec is isolation of client commands. The spec refers to this as 'multi-user support'. In our implementation this effectively means that the TPM2 commands executed over one client connection cannot effect or use objects created or loaded over another connection.

This property is great but, as always, in practice we find breaking said isolation very useful. This page documents the use-case driving a controlled break in isolation and some options for its implementation. We'll also get into how these approaches will impact clients.

For the reader to get the most out of this section is expected that they are familiar with the section titled 'Context Management' in Part 3 of the TPM specification.

## Session Continuation
Isolation places logical barriers between different connections to the `tabrmd`. An object loaded by one connection cannot be subsequently used by a command sent over a different connection. This limitation makes perfect sense for the common case: A process creates or loads an object, creates a session, uses the object and or session for some task, then the connection closes and the process shuts down. But we all know how useful it is for processes to collaborate often times passing data between themselves in a pipeline (shell pipeline processing). A strict adherence to the `isolation` property breaks this prolific software architecture.

The issue isn't quite as dire as it sounds. Generally we're concerned with TPM2 objects and sessions. If two connections want operate on the same object they have an easy work around: The TPM allows A process can save an object before it terminates. When the connection is terminated the `tabrmd` will flush the object context from the TPM, but another process can use the saved object context to reload the saved object context.

The difference between the semantics for managing object and sessions is where the issues arise: A TPM2 object may have its context saved and the object flushed from the TPM and then subsequently reloaded. Unlike a TPM2 object, a saved TPM2 session context cannot be reloaded after it has been flushed. This makes it impossible for a session created by a client connection and saved to be reloaded by another connection since the `tabrmd`, in an attempt to isolate the two connections, flushes the session context.

In practice the result is that command line tools can easily manipulate and use TPM2 objects by saving the object context and passing this data to another process where it can be reloaded and used. They cannot however do the same for sessions. Our ideal end state would be for the developer experience be identical across objects and sessions. How achievable this is will be explored in the following sections.

## Goals
Before we get too far into the details it's probably best to state our goals:
1. Simplicity: Our ideal solution will require nothing of the developer using the API beyond the expected SAPI function calls. Adding to the API to support this feature (increasing complexity) may be necessary, but it's not desirable.
2. Safe: Our solution should not allow a situation where one client has a negative impact on another. Specifically one client should not be able to prevent another from operating normally (preventing access or exhausting resources).
3. Predictable: Whatever we implement we want the experience of the developer using the feature to be consistent. Stated another way: a client application should behave the same under all but the most exceptional circumstances.

## Session Save / Load Across Connections
Focusing on our first goal of simplicity, our ideal solution would allow a session created by one connection to be loadable by another connection. Using the existing TPM2 context management functions the ideal solution would allow the creator of the session to save the context, terminate the connection, and then pass the session context to another process where it can be loaded and used in the same way as though a single connection saved and loaded the session.

The steps carried out for a fictional bash pipeline with the first command creating a session and saving the context to a file using the `-o` option, the second loading this session and doing something useful with it:
```
tpm2_session_create -o session.ctx && tpm2_session_use -i session.ctx
```

The steps this requires would be:
1. `tpm2_session_create` creates the session with `StartAuthSession`
2. `tpm2_session_create` manipulates the session into some state using various TPM2 policy commands.
3. `tpm2_session_create` saves the session context to the output file `session.ctx`
4. `tpm2_session_create` terminates and the saved session is *not* flushed. In this state the session has been abandoned but not claimed by another connection.
5. `tpm2_session_use` is passed the session context and it loads this context using the `LoadContext` command. By loading the session this connection has claimed the session abandoned in step 4.
6. `tpm2_session_use` uses the session for some purpose (authorization etc)
7. `tpm2_session_use` exits and the session is flushed

### Saving Sessions
To support the above program flow the default behavior of the tabrmd must be changed. In the current implementation all object and session contexts created over a connection are flushed when the connection is terminated. To allow another connection to use a session created by another context this behavior must be modified.

This modification should allow selective sharing of sessions. Which sessions are passed from the creating connection to the receiving session should be controlled by the creator. To support this the default behavior of the `tabrmd` should not change but instead we want a connection to be able to select some set of sessions to persist after the connection is terminated.

To accomplish this we use the `ContextSave` function. The behavior of the `tabrmd` is changed such that when a connection is closed the `tabrmd` will, before flushing an associated session context, check to see if said context was last saved by the client (by explicitly executing the `ContextSave` command) or by the `tabrmd`. If the context was last saved by the `tabrmd` then it will be flushed when the connection is closed. If the context was last saved by the client then the session context will not be flushed.

Session contexts not flushed when a connection is closed must be tracked by the `tabrmd` until they are claimed by another connection. For the purposes of this document we refer to the data structure used to hold these `paused` sessions as the `PausedSessionQueue`.

### Loading Saved Sessions
To load a session context saved in this way should be as simple as calling the `ContextLoad` command from another connection passing it the context blob. In this situation the `tabrmd` will need to intercept the `ContextLoad` command to determine whether or not the context being loaded is in the `PausedSessionQueue`. If it is then the `tabrmd` must pass ownership of this session to the connection over which is is loaded.

### Access Control
Access control in this case relies on the possession of the saved session context blob. When a session is saved and the owning connection closed the application owning the connection is responsible for passing the saved context blob to whatever entity is intended to load and use it next. This access control mechanism is outside the purview of the `tabrmd` and may rely on any number of access control mechanism. 
This relies on the saved session context blob being unique and not forgeable.

### Limitations and Drawbacks
This approach is the most seamless from the perspective of the client application. In the context of our stated goals this is the `simple` solution. It does however have a number of drawbacks enumerated in this section.

The primary issue at hand is resource exhaustion. Limited space in the TPM means a finite number of sessions may be in existence or loaded at any time. For most TPMs (in my experience) there is a maximum number of 64 sessions that can be in existence at a time. Once this maximum is reached no new sessions may be created until another is flushed. If the `tabrmd` allows sessions to live on beyond the end of a connection for any reason then we've poked a hole in its ability to predictably manage these scarce resources.

If we assume that the `tabrmd` can limit the number of client connections and the number of sessions allocated to each then the 64 available sessions could be evenly divided. By adding the ability for a connection to cause sessions to live on past the end of the connection we make it possible for sessions to be intentionally left resident and never reclaimed. This would require a process to create a connection, create a session, save the session and then close the connection repeatedly without having another connection claim / load it. If done repeatedly the eventual outcome will be the exhaustion of session space in the TPM which will prevent all new connections from creating sessions.

### Proposed Workarounds
The limitations of this approach cannot be prevented completely. So long as TPM2 devices have a hard limit on the number of sessions (saved or loaded) they cannot be eliminated, though the impact can be mitigated. This section identifies two possible mitigations.

#### Least-Recently-Saved Session Flushing
Least recently used (LRU) algorithms are common in operating system caching systems (https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_Recently_Used). In this work-around the algorithm would beadapted to identify the least recently continued session and when an upper bound of unclaimed sessions is reached the `tabrmd` would flush it freeing up space. This space would then be used by the newly saved / continued session.

This work-around isn't ideal though. Used in this way the LRU may cause situations in which failures become very difficult to reproduce since they are caused by unpredictable interactions between clients (some may even cause this a "Heisenbug": https://en.wikipedia.org/wiki/Heisenbug). Such a situation could be caused by a user attempting to continue a session from one process to another but during the time between one connection saving a session and another loading it the actions of some other processes may cause the LRU to flush the session thus causing a failure. A misbehaving process could increase the likelihood of this situation by rapidly and repeatedly creating a connection, creating a session, saving the session and closing the connection.

#### Administrative Interface
If the unpredictability of the LRU approach is undesirable then hard limits may be the only acceptable option.
As discussed in the `Limitataions and Drawbacks` section, hard limits could cause resource exhaustion preventing use of sessions.
In situations where the space allocated for abandoned sessions has been exhausted no connections will be able to continue sessions.

This workaround proposes the use of an administrative interface to allow the flushing of all sessions that have been abandoned and not claimed on demand.
The shortcomings of this approach is the requiremnet for administrative intervention.
In a scenario where these resources have been exhausted users will be left without use of TPM sessions until an administrator is able to intervene.
