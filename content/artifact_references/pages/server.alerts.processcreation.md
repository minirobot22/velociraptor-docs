---
title: Server.Alerts.ProcessCreation
hidden: true
tags: [Server Event Artifact]
---

This artifact alerts when a process was detected with the artifact 'Windows.Detection.ProcessCreation' (which is a client_event artifact that needs to be enabled first).


```yaml
name: Server.Alerts.ProcessCreation
description: |
   This artifact alerts when a process was detected with the artifact 'Windows.Detection.ProcessCreation' (which is a client_event artifact that needs to be enabled first).

author: Jos Clephas - @DfirJos

type: SERVER_EVENT

parameters:
  - name: SlackToken
    description: The token URL obtained from Slack/Teams/Discord (or basicly any communication-service that supports webhooks). Leave blank to use server metadata. e.g. https://hooks.slack.com/services/XXXX/YYYY/ZZZZ

sources:
  - query: |
        LET token_url = if(
           condition=SlackToken,
           then=SlackToken,
           else=server_metadata().SlackToken)

        LET hits = SELECT * from watch_monitoring(artifact='Windows.Detection.ProcessCreation')

        SELECT * FROM foreach(row=hits,
        query={
           SELECT EventData.CommandLine, EventData, Hostname, ClientId, Url, Content, Response FROM http_client(
            data=serialize(item=dict(
                text=format(format="Alert - Command detected '%v' on system %v with client Id %v. Syslog timestamp: %v ",
                            args=[EventData.CommandLine, Hostname, ClientId, Timestamp])),
                format="json"),
            headers=dict(`Content-Type`="application/json"),
            method="POST",
            url=token_url)
        })

```
