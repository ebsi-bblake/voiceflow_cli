## Explaining Logux frame shape and key operation patterns

# Big picture

 The Logux integration is a small WebSocket client layer used for two jobs:

 1. Read the Voiceflow catalog — xyops/voiceflow/logux/index.ts
 2. Perform realtime mutations — rename projects, create folders, create secrets

 All operations follow roughly:

 ```text
   open WebSocket
     → send Logux connect frame
     → receive connected
     → subscribe to a workspace/assistant channel
     → wait for matching synced frame
     → send mutation or collect catalog actions
     → wait for operation-specific completion
     → close socket exactly once
 ```

 1. Logux wire frame shape

 Frames are JSON arrays rather than objects:

 ```ts
   [
     "sync",
     syncID,
     action,
     {
       id: actionID,
       time: actionTime
     }
   ]
 ```

 Connection starts with:

 ```ts
   [
     "connect",
     4,
     origin,
     0,
     {
       token,
       subprotocol: "1.9.0"
     }
   ]
 ```

 The 4 is the Logux protocol version. origin identifies the client/session.

 2. Catalog synchronization

 syncCatalog() opens a socket and sends:

 ```ts
   [
     "sync",
     requestID,
     {
       channel: "workspace/{workspaceID}",
       type: "logux/subscribe",
       since: { id: "0", time: 0 }
     },
     {
       id: nextActionID(),
       time: nextActionTime()
     }
   ]
 ```

 The server then emits action frames such as:

 ```ts
   [
     "sync",
     ...,
     {
       type: "project.CRUD:REPLACE",
       payload: {
         values: [...]
       }
     }
   ]
 ```

 frames.ts:

 - Parses only text WebSocket messages.
 - Rejects malformed JSON implicitly by ignoring it.
 - Enforces:
     - 1 MiB maximum frame size
     - 8 MiB total incoming data
     - 100,000 catalog rows
 - Accepts only explicitly requested action types.
 - Accumulates rows until every requested type has been observed.
 - Converts server errors into OperationFaults.
 - Closes the connection through a single settlement path.

 That is the read-side pattern: subscribe, filter known action types, collect a bounded
 snapshot, resolve when complete.

 3. Project rename flow

 rename-project.ts is the important mutation pattern.

 ### Step 1: Connect

 It creates:

 ```ts
   const origin = `${creatorID}:${uuid}:${uuid}`;
   const subscriptionID = randomNumber();
   const mutationSyncID = subscriptionID + 1;
 ```

 The two sync IDs are deliberately different:

 - subscriptionID identifies subscription readiness.
 - mutationSyncID identifies the rename mutation.

 ### Step 2: Subscribe

 After receiving connected, it sends:

 ```ts
   {
     channel: `workspace/${workspaceID}`,
     type: "logux/subscribe"
   }
 ```

 It waits specifically for:

 ```ts
   ["synced", subscriptionID]
 ```

 Only then does it send the mutation.

 ### Step 3: Send the mutation

 The rename action is:

 ```ts
   {
     type: "assistant.PATCH_ONE",
     payload: {
       id: projectID,
       patch: { name },
       context: { workspaceID }
     },
     meta: {
       origin,
       actionID: createUUID()
     }
   }
 ```

 This is wrapped in a "sync" frame using mutationSyncID.

 ### Step 4: Distinguish acknowledgement from observation

 This is the key correctness rule:

 ```ts
   frame[0] === "synced" && frame[1] === mutationSyncID
 ```

 means the mutation request was acknowledged.

 A separate broadcast like:

 ```ts
   project.CRUD:PATCH
 ```

 only proves that a matching state update was observed. It is not used as the mutation
 acknowledgement.

 That distinction prevents the migration from racing ahead because of:

 - a stale broadcast,
 - a broadcast from another client,
 - an unrelated processed event,
 - an update that arrived before the current mutation completed.

 The code then resolves on the matching mutation synced frame and lets
 confirmProjectRename() re-read the catalog for durability.

 4. Secret creation flow

 create-secret.ts uses the same lifecycle but has a different completion signal.

 It:

 1. Connects with a redacted diagnostic copy of the frame.
 2. Subscribes to assistant/{assistantID}.
 3. Waits for the subscription’s matching synced.
 4. Sends secret.CREATE_ONE_STARTED.
 5. Waits for:

 ```ts
   action.type === "secret.CREATE_ONE_DONE"
   action.meta.actionID === originalActionID
 ```

 The secret value is sent in the actual WebSocket frame but is never included in logs.
 Diagnostics log only:

 - frame type
 - sync ID
 - action type
 - action ID
 - processed ID

 Secrets are created sequentially by createProjectSecrets():

 ```ts
   secrets.reduce(
     (pending, secret) =>
       pending.then(() => createSecret(auth, assistantID, secret)),
     Promise.resolve(),
   )
 ```

 That avoids concurrent mutations against the same assistant.

 5. Folder creation

 create-folder.ts is an older/simpler version of the same pattern:

 ```text
   connect
   → subscribe workspace
   → wait for subscription synced
   → send workspace-folder.CREATE_ONE_STARTED
   → find workspace-folder.CREATE_ONE_DONE by actionID
   → extract folder ID
 ```

 Unlike project rename, it does not use the mutation sync ID as its completion
 criterion. It relies on the completion action’s meta.actionID.

 6. How migration uses these operations

 execute_migration/index.ts orchestrates them:

 ```text
   load destination catalog
   → find exact project collision
   → rename collision through Logux
   → re-read catalog until rename is durable
   → import project
   → create configured secrets sequentially
 ```

 The important safety gate is:

 ```ts
   await renameProject(...)
   await confirmProjectRename(...)
   await importVersion(...)
 ```

 Import cannot begin merely because the rename request was sent. The renamed project
 must be visible in a fresh catalog read.

 Main patterns worth preserving

 - Correlate every operation with unique IDs.
 - Wait for the specific synced ID you created.
 - Match completion events by actionID.
 - Treat broadcasts as observations, not acknowledgements.
 - Use one idempotent settlement path.
 - Always enforce timeouts and close the socket.
 - Redact credentials and secret values from diagnostics.
 - Bound incoming frame size and row count.
 - Keep mutations sequential when ordering matters.
