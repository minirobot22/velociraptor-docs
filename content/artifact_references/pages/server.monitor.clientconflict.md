---
title: Server.Monitor.ClientConflict
hidden: true
tags: [Server Event Artifact]
---

Sometimes the Velociraptor client is installed into a VM template
image with an existing write back file. In this case each VM
instance will start the client with the same client id.

When clients connect to the server multiple times, the server will
reject one with the HTTP 409 Conflict response.

This artifact will also force conflicting clients to rekey
themselves. Clients will generate a new client id and reconnect with
the server, saving their new keys into their write back files.


```yaml
name: Server.Monitor.ClientConflict
type: SERVER_EVENT
description: |
  Sometimes the Velociraptor client is installed into a VM template
  image with an existing write back file. In this case each VM
  instance will start the client with the same client id.

  When clients connect to the server multiple times, the server will
  reject one with the HTTP 409 Conflict response.

  This artifact will also force conflicting clients to rekey
  themselves. Clients will generate a new client id and reconnect with
  the server, saving their new keys into their write back files.

sources:
  - query: |
      SELECT
        collect_client(client_id=ClientId,
            artifacts="Generic.Client.Rekey", env=dict())
      AS NewCollection
      FROM watch_monitoring(artifact="Server.Internal.ClientConflict")

```
