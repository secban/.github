# SecBan

SecBan is a intrusion-detection framework that bans malicious hosts at the firewall level. It is heavily inspired by fail2ban, but takes a more granular architecture.

## Overview

Instead of a single component, SecBan is split into several services. The two main ones are:
- `secban-agent`: This is the core component. It listens for report/ban requests on a tcp address and/or a unix socket and trigger the firewall block.
- `secban-log-monitor`: Is the part that reads logs and match requests against bad patterns, calling the agent with findings

One of the main benefit of having a separate `agent`, which accept REST requests is that you don't have to rely on parsing logs, which can sometime be brittle and rigid. The idea here is that for web traffic, you can create more dynamic rules directly within your app, which has already access to more business logic than what you can gather from access logs. Depending on your usecase and setup, you might even only need the agent.

## Extendable actions

The `agent` only does one thing, keeping track of reported "offenders" and blocking them when a threshold is met. It is easily extendable through custom actions/scripts, or by dispatching events through unix sockets.
The `secban-mailer` service, for example, can be used to send an email when a ban occurs.