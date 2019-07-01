---
RFC: RFCXXXX
Author: Travis Plunk
Status: Draft
SupersededBy:
Version: 1.0
Area: Engine
Comments Due: 08/15/2019
Plan to implement: Yes
---

# Title

Remote ScriptBlock Logging

## Description

This is an addition to the existing ScriptBlock creation logging.
The goal is to make it easier to configure to collect the logs centrally.
This feature will be experimental.

## Motivation

As a administrator of machines which run PowerShell (machine admins),
I can easily log script blocks which PowerShell executes to systems such as Azure Log Analytics or Splunk (log systems),
so that I can analyze PowerShell activity on all the machines I run.

## User Experience

This example assumes you have registered the key to communicate
with Azure Log Analytics with PowerShell Secrets with the name `myKeyName`.

**Note:** `myKeyName` is a secret register using the system added in the [Secrets Management RFC][secrets-rfc]

### First time registration

```powershell
Register-PSScriptBlockLogger -Provider AzureLogAnalytics -KeyName myKeyName -Properties @{
    WorkspaceId = ‘myworkspaceid’
}
```

```output
Provider: AzureLogAnalytics
Properties: {WorkspaceId="myworkspaceid"}
KeyName: myKeyName
```

### Register when something is already registered

```powershell
Register-PSScriptBlockLogger -Provider AzureLogAnalytics -KeyName myKeyName -Properties @{
    WorkspaceId = ‘myworkspaceid’
}
```

```output
Error: A provider is already registered.  User -Force to overwrite the current provider settings.
```

### Unregister

```powershell
Unregister-PSScriptBlockLogger
```

Upon success, there would be no output.

### Get current registration

Note: this example assumes that the provider was already register as in [First time registration](#first-time-registration)

```powershell
Get-PSScriptBlockLogger
```

```output
Provider: AzureLogAnalytics
Properties: {WorkspaceId="myworkspaceid"}
KeyName: myKeyName
```

## Specification

### Assumptions

1. The transport to the Logging system is secure.
   Secure includes:
   - Encrypted
   - Authenticated
1. The machine admin is willing and authorized to send all ScriptBlocks to the logging system.

### Registration

Only one logging provider can be registered at a time.
There will be two options for registration,
which are explained below:

#### Registration cmdlet

There will be a cmdlet to register a logging provider.
`Register-PSScriptBlockLogger` will allow you to register a provider.
It will update the Computer-Wide PowerShell configuration file to persist this as a policy.

#### Policy

There will be a new policy with three settings:

- ScriptBlockLoggingProvider - The name of the provider
- ScriptBlockLoggingProviderKeyName - The name of [secret][secrets-rfc] storing the key used to access the providers.
- ScriptBlockLoggingProviderProperties - Properties needed by the provider, persisted as JSON.

### Collecting and Sending Data

We will add code in `CompiledScriptBlock.LogScriptBlockCreation`
which adds the data to log into a FIFO structure (send queue).
A background task will be started, which will process the data.
The background task data processing will have two responsibilities:

1. Batching and sending the data.
1. Ensuring that the data in the send queue does not grow too large.

#### Background Task - Batching and sending

The background task will batch data into group and
send data based on specification from the logging provider.
The task will have a timeout, if a batch is not full by the end of the timeout,
it will send the current batch.
This timeout will reset each time a new item is added to the batch.

#### Background Task - Limiting the size of the send queue

The task will also monitor the number of items in the send queue
and delete items if the number of items exceeds a predefined limit.

### Data to send

| Name                  | Type     | Description                                                                            |
|-----------------------|----------|----------------------------------------------------------------------------------------|
| ScriptBlockText       | string   | The text of the script block (may be truncated, based on field size limit of provider) |
| UtcTime               | datetime | The time on the local machine which the ScriptBlock was started                        |
| ScriptBlockHash       | string   | A hash of the script block                                                             |
| File                  | string   | If applicable, the name of the file the script block came from                         |
| ParentScriptBlockHash | string   | A hash of the script block which, ran this script block                                |
| Computer              | string   | The name of the computer which ran the script block                                    |
| ProcessID             | string   | The process id which ran the script block                                              |
| RunspaceId            | double   | The id of the PowerShell Runspace which ran this ScriptBlock                           |
| Runpacename           | string   | If it exists, the PowerShell Runspace name, which ran this ScriptBlock                 |
| User                  | string   | The user which ran this ScriptBlock                                                    |
| CommandName           | string   | The name of the first command in the block                                             |
| CompressedScriptBlock | string   | If `ScriptBlockText`is truncated, a Britoli compressed version base64 coded.           |
| BatchOrder            | double   | The order we sent this item inside a batch, helps display event in the correct order   |

### Onboarding

Assuming you already have an Azure Subscription,
a cmdlet will be provided, using the `az` module,
to walk you through creating an Azure Log Analytics workspaces and creating a deployment script.

### API used to send data

https://docs.microsoft.com/en-us/azure/azure-monitor/platform/data-collector-api
This API is in preview and is subject to change.
This is one of the reasons for the feature being experimental.

## Alternate Proposals and Considerations

### Original RFC

Note, that an RFC was filed for this by @rhysjtevans.
See [#106](https://github.com/PowerShell/PowerShell-RFC/pull/106).

Support for unencrypted communications has been delayed
until [Encrypting the ScriptBlock](#encrypting-the-scriptblock) can be implemented.

Initially this feature will only support ScriptBlock creation logging.
Please see [Additional logging](#additional-logging).

## Future Work

### Splunk provider

We would like to integrate a provider for sending data to Splunk.
We could do this work in the future or someone else could contribute this.

### Encrypting the ScriptBlock

For this to be effective, the log services would need support.
We can consider this when they have support.

### Using existing credentials

We will investigate using a logged in Microsoft Service Identify or
Azure Active Directory user to send data.

### Using existing installed Azure Log Analytics agent

We continue to investigate how to make it easier to
use an existing installed Azure Log Analytics agent to send this data.

[secrets-rfc]: https://github.com/PowerShell/PowerShell-RFC/pull/208

### Additional logging

The current plan is to log ScriptBlock creation.
Additional logging being consider include:

- ScriptBlock execution
- Transcription

Either of these would take up significantly more bandwidth.
We think this feature should be tried before we expand the feature.
