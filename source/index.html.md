---
title: WeiWatchers Documentation

<!-- language_tabs: -->
  <!-- - shell -->

toc_footers:
  - <a href='mailto:support@smartcontract.com'>Need Help?</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to the WeiWatchers API documentation. WeiWatchers is a service which turns changes in Ethereum blockchains into push notifications for other applications. WeiWatchers can report on Ethereum event logs or account balance changes. Additionally, it can be configured to run on any Ethereum chain, public or private.

If you want to view the source code or build the image locally, you can do so on [GitHub](https://github.com/oraclekit/wei_watchers).  

## Installation

```shell
docker pull smartcontract/wei_watchers
docker run -it --env-file=.env smartcontract/smartoracle rake watcher:create
```

> Which base URL would you like notifications to be sent to?

```shell
http://localhost:3000/wei_watchers_base/
```

> Watcher Username:               RANDOM_KEY_1 ...

```shell
docker run -t --env-file=.env smartcontract/wei_watchers
```

### Prerequisites

- Docker installed
- a running instance of Postgres, 9.4 or above
- a running instance of Ethereum with the JSON-RPC interface enabled

### Install

First set up your configuration, by creating a `.env` file, that has at least the variables set in the [.env.example](https://github.com/oraclekit/wei_watchers/blob/master/.env.example).

Once you have set up your database and configuration, and you are ready to initialize WeiWatchers.

Run the watcher creation command and provide the URL where you would like push notifications to be sent.

Once a watcher is successfully initialized two sets of credentials will be printed out. The first credentials are used to create new event or account subscriptions on the watcher via API. The second is the HTTP basic authorization credentials that will be used by Wei Watchers when pushing updates out to applications.

<br/>

To have a managed instance spun up for you, contact us at [support@smartcontract.com](mailto:support@smartcontract.com).


## Authentication

Wei Watchers currently uses HTTP basic authorization for authentication. There are two different types of credentials used for interacting with Wei Watchers, Watcher Credentials are used to send updates to Wei Watchers, and Notification Credentials are used by Wei Watchers to send updates to applications. The two are intentionally kept distinct, but can be made to match for convenience sake.

Watchers and their credentials are currently created and managed via the command line only. Once created, Watcher information and settings are available via API accesss.

### Watcher Credentials

```shell
curl -u watcherKey:watcherSecret
  http://localhost:3174/status
```

Watcher credentials are used to authenticate with Wei Watchers and create new notification subscriptions. Watcher credentials are required and automatically generated for you upon creating a new Watcher. They can be updated, but never removed.

Watcher credentials are used to access all API endpoints and update settings. Wathcers are set up as a multitenant system; Watcher's cannot access or update information for other Watchers.

### Notification Credentials

Notification credentials are used by Wei Watchers to authenticate with the endpoints receiving the notifications. Notificaiton credentials are not required, although they are automatically generated upon creating a new Watcher. They can be updated or removed.

# Assignments

## Anatomy of an Assignment
Assignments are the core of the Smart Oracle model, they are the specifications of work given to an oracle. The main pieces of an assignment consist of the [adapter](#adapter-type-and-parameters), the [scheduling](#scheduling), and the [payment specification](#payment). Optionally, an assignment can have a description and signatures from the parties involved.

### Adapter Type and Parameters
Every assignment has a type, which specifies the kind of adapter the oracle will use to perform the work requested. Each different type of adapter requires different parameters, some passed on the blockchain, some passed off chain. The parameters each adapter will accept are specified ahead of time by the adapter's creator.

The current Smart Oracle ships with several built in adapters: `ethereumBytes32` for writing strings into Ethereum, `ethereumInt256` for writing signed integers into Ethereum, `ethereumUint256` for writing unsigned integer into Ethereum, `httpGetJSON` for retrieving JSON across the web, and `bitcoinComparisonJSON` for releasing Bitcoin escrow based on the values of JSON APIs. For more information on setting up custom adapters, see the section on [creating an adapter](/#adapter-integration).

### Subtasks
```json
{
  "adapterType": "ethereumBytes32",
  "adapterParams": {
    "address": "0x34d946ab16079a97976e006079efe422b1fe1905",
    "method": "8b147245"
  }
}
```
Subtasks are a series of steps taken by the oracle to complete an assignment. Each time an assignment is updated it processes its pipeline of subtasks, each of which is handled by an [adapter](#adapter-type-and-parameters). The subtask's initial configuration, along with any data passed from earlier subtasks in the pipeline are passed to the adapter for each update. The adapter's output for the subtask is passed to the next subtask, until the final subtask's output determines the values of the assignment's latest [snapshot](#snapshots).

Parameter | Type | Description
---- | ----- | --------
adapterType | string | identifies the type of work to be done
adapterParams | object | a JSON object meeting the requirements of the adapter's schema, setting the initial configuration for the adapter

### Scheduling
Some oracle services are needed on a scheduled basis. For these use cases, the Smart Oracle a simple way to schedule tasks based on the [Cron](https://en.wikipedia.org/wiki/Cron) scheduling format, or specify specific times to run at.

Assignments that do not require a schedule, instead work on demand, can skip the schedule or offer a combination of both on demand updates and scheduled updates.

All assignments require a end time and allow for an optional start time, defaulting to the assignment start time. Times are specified using Unix timestamps(UTC).

```json
{
  "minute": "1",
  "hour": "17",
  "dayOfMonth": "*",
  "monthOfYear": "*",
  "dayOfWeek": "*",
  "startAt": "1477941966",
  "endAt": "1793474734",
  "runAt": ["1518726087", "1802722889"]
}
```

Parameter | Type | Description
---- | ----- | --------
minute | string | cron style minute listing for how often to run recurring snapshots
hour | string | cron style hour listing for how often to run recurring snapshots
dayOfMonth | string | cron style day of month listing for how often to run recurring snapshots
monthOfYear | string | cron style month of year listing for how often to run recurring snapshots
dayOfWeek | string | cron style day of week listing for how often to run recurring snapshots
startAt | string | a unix timestamp for when to start the recurring snapshot schedule. Optional defaults to the time the assignment is created.
endAt | string | a unix timestamp for when to end the recurring snapshot schedule
runAt | array | an array of unix timestamps for discrete update times


### Payment
```json
{
  "currency": "ETH",
  "perDay": "0",
  "perRequest": "1000000000000000"
}
```
Space on blockchains is limited and thus costly to utilize. On the other side of the oracle, API calls to protected APIs are often behind pay walls. Oracle services by their nature come with expenses. For this reason, the Smart Oracle platform has a way to specify service prices for each adapter.

Prices are generally set per request made by the oracle; This can be prepaid for scheduled services, or paid per request for on demand services. Additionally, prices can be set based on the duration of required availability of the oracle.

The currency is configurable based on the networks that the oracle operates in. Currency amounts are always specified in the lowest possible denomination of the currency(Satoshis, Wei, etc.).

## Create
```shell
curl -u apiKey:apiSecret -X POST -H 'Content-Type: application/json'
  -d '{"assignment":{"subtasks":[{"adapterType": "httpGetJSON", "adapterParams": {"endpoint": "https://bitstamp.net/api/ticker/", "fields": ["last"]}},{"adapterType":"ethereumBytes32"}]}, "schedule":{"endAt":"1478028219","hour":"0","minute":"0"}},"version":"1.0.0"}'
  http://localhost:3174/assignments
```

> JSON response:

```json
{
  "assignmentHash": "b2e55902bb7728871fa69f503007577ef8a1ae449f486b5c0aaf644661d216d1",
  "signature": "1cb6770f6977710c7c3e0d336d3d2244a0d276056029cb83f414d8641ef412218d2a36df9ccde31ee50310d9eef098fa153b34ad9075d02737243a59c5dbd6d357",
  "xid": "561b78af-e163-4972-9d4a-5dc15e02d977"
}
```

A `POST` to `/assignments` will return the `XID`, or external ID, which is used to identify the assignment in the future. The response also includes a hash of the assignment specified, and a signature of that hash to attest to the oracle's acceptance of the assignment.

Parameter | Type | Description
---- | ----- | --------
subtasks | array | an ordered list of the assignment's [subtasks](#subtasks)
schedule | object | an [assignment schedule object](#scheduling)
version | string | specify the version of the [assignment spec](https://github.com/smartoracles/spec), use "1.0.0" for the latest version


<aside class="success">
Make sure to grab the XID to refer to the assignment in the future. The assignment hash is intentionally not used in the future, as the XID is not easily linkable with the assignment.
</aside>


## Show

```shell
curl -u apiKey:apiSecret http://localhost:3174/assignments/f0577c3e-9e5a-4840-9c9b-37d326c3d2e3
```

> JSON response:

```json
{
  "adapterType": "ethereumBytes32JSON",
  "endAt": "1480720191",
  "parameters": {
    "fields": ["last"],
    "endpoint": "https://bitstamp.net/api/ticker/"
  },
  "snapshots": [
    {
      "description": "Blockchain Record: 0x865167b8e74daec5e0f2faf4975b7fe21d791b83d7c6ba10ece7e4d0b193461e",
      "descriptionURL": "https://testnet.etherscan.io/tx/0x865167b8e74daec5e0f2faf4975b7fe21d791b83d7c6ba10ece7e4d0b193461e",
      "details": {
        "value": "726.60"
      },
      "summary": "Assignment \"1\" updated its value to \"726.60\".",
      "value": "726.60",
      "xid": "0x865167b8e74daec5e0f2faf4975b7fe21d791b83d7c6ba10ece7e4d0b193461e"
    }
  ],
  "startAt": "1478128191",
  "status":"in progress",
  "xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3"
}
```
A coordinator can check the state of one of their assignments and get a list of all the related snapshots.

Parameter | Type | Description
---- | ----- | --------
adapterType | string | identifies the type of work to be done
endAt | string | specify the time the assignment will run until (Unix timestamp)
parameters | object | a JSON object meeting the requirements of the adapter's schema
snapshots | array | an list of associated snapshot objects
startAt | string | specify the time the assignment is/was scheduled to start at (Unix timestamp)
status | string | assignment's current execution status
xid | string | a unique external ID to identify the assignment

### Snapshots

Parameter | Type | Description
---- | ----- | --------
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by


# Snapshots

## Create
```shell
curl -u apiKey:apiSecret -X POST
  http://localhost:3174/assignments/f0577c3e-9e5a-4840-9c9b-37d326c3d2e3/snapshots
```

> JSON response:

```json
{
  "assignmentXID": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3",
  "description": "Blockchain record: 0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
  "descriptionURL": "https://etherscan.io/tx/0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
  "details": {
    "source": "bitstamp.net",
    "txid": "0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
    "value": "1035.03"
  },
  "summary": "The oracle value was updated to \"1035.03\"",
  "value": "1035.03",
  "xid": "0799f829-15e8-475f-8f3e-a55488713da0"
}
```


A `POST` to `/assignments/XID/snapshots` will always return the `XID` of a snapshot. Depending on what steps are required of the assignment, it may return all of the snapshot immediately, or if the details take time to compute the results of the snapshot will be retrievable with the assignment's `XID`.

Parameter | Type | Description
---- | ----- | --------
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by

## Show

```shell
curl -u apiKey:apiSecret http://localhost:3174/snapshots/0799f829-15e8-475f-8f3e-a55488713da0
```

> JSON response:

```json
{
  "assignmentXID": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3",
  "description": "Blockchain record: 0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
  "descriptionURL": "https://etherscan.io/tx/0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
  "details": {
    "source": "bitstamp.net",
    "txid": "0x2d8b24bc521a7fb02c1eb3e157296517f628646f6b5b8ee8952034fa2ca7f385",
    "value": "1035.03"
  },
  "summary": "The oracle value was updated to \"1035.03\"",
  "value": "1035.03",
  "xid": "0799f829-15e8-475f-8f3e-a55488713da0"
}
```

To retrieve the details of a snapshot, you can issue a `GET` request to `/snapshots/XID`.

Parameter | Type | Description
---- | ----- | --------
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by



# Coordinator Updates

Once an assignment is created, it will automatically run and update itself based on its schedule and the logic in the adapter. Updates cannot be fed in by the coordinator, but the coordinator can receive push notifications whenever the assignment is updated. Simply set the URL of the assignment's coordinator to receive push notifications.

All push notifications are authorized with the same credentials used to create the assignment.

## Create Assignment Oracle
```json
{
  "oracle": {
    "address": "0x72c8379f845bb3cb30e02ef1feb84742debc1efb",
    "jsonABI": "contract Oracle {\n  bytes32 currentValue;\n  address creator;\n\n  function Oracle() {\n    creator = msg.sender;\n  }\n\n  function update(bytes32 newCurrent) {\n    if (msg.sender != creator) return;\n    currentValue = newCurrent;\n  }\n\n  function current() constant returns (bytes32 current) {\n    return currentValue;\n  }\n\n  function () constant returns (bytes32 current) {\n    return currentValue;\n  }\n}",
    "readAddress": "9fa6a6e3",
    "solidityABI": "contract Oracle{function Oracle();function update(bytes32 newCurrent);function current()constant returns(bytes32 current);}"
  },
  "xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3"
}
```

`POST` to `/assignments/:xid/instructions`

If an assignment needs some on-chain setup, it cannot immediately respond with all of its integration details on creation, it needs to wait for the blockchain confirmations before. Once the confirmations needed for the assignment set up have occured, the oracle will push integration instructions to the coordinator.

Parameter | Type | Description
---- | ----- | --------
address | string | Ethereum address location
jsonABI | string | a stringified version of the JSON ABI for the contract
readAddress | string | the hash for the read function of the Ethereum contract
solidityABI | string | the Solidity ABI to include in a contract using this oracle
xid | string | the XID of the related assignment


## Create Assignment Snapshot

```json
{
  "assignmentXID": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3",
  "description": "Blockchain ID: 0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046",
  "descriptionURL": "https://testnet.etherscan.io/tx/0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046",
  "details": {
    "current": "10000000034567123",
    "total": "98700000035511224"
  },
  "status": "in progress",
  "summary": "Assignment 'f0577c3e-9e5a-4840-9c9b-37d326c3d2e3' updated its value to \"1,000,000.00\"",
  "value": "1,000,000.00",
  "xid": "0x5803d6bb728b002d5a9aedc2eebec404fcbc2e2966f5e81abe297995b2980046"
}
```

`POST` to `/assignments/:assignment_xid/snapshots`

Each time the assignment is updated it creates a snapshot. A snapshot is the current status of the assignment. Whenever a snapshot is created, it is pushed to the coordinator. Further information is provided in machine readable form in the `details` section, in various formats depending on the adapter. Information is provided in human readable form in the `summary` and `description` fields, as well as the `descriptionURL`.

Parameter | Type | Description
---- | ----- | --------
assignmentXID | string | the XID of the related assignment
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by

<aside class="success">
Snapshot IDs are unique, you should never receive duplicate snapshots.
</aside>

## Update Assignment

```json
{
  "signatures": ["30450221009f4e3e0ab2ede2ca57a03ea67f6d16641568b214e38b650db95cd3490b3f213902202b8ba93235c6cafa4f8a643c72977427e8fde95a2e8010cb3336125182d62cef"],
  "status": "completed",
  "xid": "f0577c3e-9e5a-4840-9c9b-37d326c3d2e3"
}
```

`PATCH` to `/assignments/:xid`

When the assignment is finished, a notification will be pushed to the coordinator. If any signatures were needed from the oracle, for things like releasing Bitcoin escrow, a signature is returned in addition to a status update.

Parameter | Type | Description
---- | ----- | --------
signatures | array(string) | if a signature is required, like for Bitcoin escrow, it returns a signature for the outcome that was determined
status | string | Either "completed" or "failed"
xid | string | the XID of the related assignment


# Adapter Integration

The Smart Oracle image ships with functionality out of the box to connect Ethereum contracts and Bitcoin escrow to JSON APIs(the lingua franca of the web). But the real power of the Smart Oracle platform lies in its abilitiy to be extended.

Adapters can be configured with the Smart Oracle core to add functionality. Whether it's a different data format like XML, or special computation, the Smart Oracle makes it possible to further extend off-chain capabilities via adapters.

### APIs

Adapters integrate with the core in a service oriented model, so they can run locally, next to the core or on remote servers. The minimum integrations an adapter needs to support are based on whether the adapter will push information or the core will pull. The APIs needed to create a custom adapter are listed below.

### Oracle Schema

In order to make an adapter's input and output predictable, adapters need to specify schemas. Schemas are created using a JSON Schema for expected prerequisites and on/off-chain inputs/outputs. See the [specification here](https://github.com/smartoracles/spec).


## Create Assignment

__*Required*__: `POST` to adapter path `/assignments`

The action used for an oracle to pass an assignment over to an adapter.

The expected response should include:

Parameter | Type | Description
---- | ----- | --------
xid | string | the unique identifier to associate an assignment with
endAt | string | a timestamp in Unix Timestamp(UTC) format.
data | object | a JSON object containing all information specified in the adapter's schema


## Create Snapshot _(Pull)_

`POST` to adapter path `/assignments/:assignment_xid/snapshots`

The action used for an oracle to pass an assignment over to an adapter.

Parameter | Type | Description
---- | ----- | --------
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
fulfilled | boolean | marks whether the snapshot has been completed or not
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by

### Unfulfilled Snapshots

Creating a snapshot may require more time than you are willing to leave a request hanging for. Snapshots that are requested via the pull style can be marked as unfulfilled upon creation, and fulfilled at a later time.

In order to fulfill an unfulfilled snapshot you must implement the update snapshot integration.


## Delete Assignment

`DELETE` to `/assignments/:xid`

Used to indicate the end of an assignment.

Parameter | Type | Description
---- | ----- | --------
status | string | optional string to specify the final state of the assignment
xid | string | identifier of the assignment


## Create Snapshot _(Push)_

`POST` pushed to the core path `/snapshots`

__This is an integration that originates in the adapter and is pushed from the adapter to the core.__

A pushed snapshot is automatically marked as fulfilled.

Parameter | Type | Description
---- | ----- | --------
assignmentXID | string | identifier for the associated assignment
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by

## Update Snapshot _(Push)_

`PATCH` pushed to the core path `/snapshots/:xid`

__This is an integration that originates in the adapter and is pushed from the adapter to the core.__

Only unfulfilled snapshots can be updated. When a snapshot is updated it is automatically marked as fulfilled.

Parameter | Type | Description
---- | ----- | --------
assignmentXID | string | identifier for the associated assignment
description | string | a detailed human readable description of the snapshot
descriptionURL | string | a supporting URL relating to the update
details | object | a JSON object of extra supporting information returned by the adapter
status | string | a description of the assignment's current status
summary | string | a short human readable summary of the snapshot
value | string | the latest value returned by the adapter
xid | string | a unique external ID to identify the snapshot by
