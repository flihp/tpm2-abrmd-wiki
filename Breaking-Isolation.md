One of the core TAB/RM functions outlined in the TCG spec is isolation of client commands. The spec refers to this as 'multi-user support'. In our implementation this effectively means that the TPM2 commands executed over one client connection cannot effect or use objects created or loaded over another connection.

This property is great but, as always, in practice we find breaking said isolation very useful. This page documents the use-case driving a controlled break in isolation and some options for its implementation. We'll also get into how these approaches will impact clients.

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
