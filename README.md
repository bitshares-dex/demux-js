# demux-js [![Build Status](https://travis-ci.org/EOSIO/demux-js.svg?branch=develop)](https://travis-ci.org/EOSIO/demux-js)

Demux is a backend infrastructure pattern for sourcing blockchain events to deterministically update queryable datastores and trigger side effects. This library serves as a reference implementation of that pattern for use with Node applications.

## Installation


```bash
# Using yarn
yarn add demux

# Using npm
npm install demux --save
```
## Overview

Taking inspiration from the [Flux Architecture](https://facebook.github.io/flux/docs/in-depth-overview.html#content) pattern and [Redux](https://github.com/reduxjs/redux/), Demux was born out of the following qualifications:

1. A separation of concerns between how state exists on the blockchain and how it is queried by the client front-end
1. Client front-end not solely responsible for determining derived, reduced, and/or accumulated state
1. Ability for blockchain events to trigger new transactions, as well as other side effects outside of the blockchain
1. The blockchain as the single source of truth for all application state

### Separated Persistence Layer

Storing data in indexed state on blockchains can be useful for three reasons: decentralized consensus of computation results, usage of state from within other blockchain computations, and for retrieval of state for use in client front-ends. When building more complicated front-ends, you run into a few problems when retrieving directly from indexed blockchain state:

* The query interface used to retrieve the indexed data is limited. Complex data requirements can mean you either have to make an excess number of queries and process the data on the client, or you must store additional derivative data on the blockchain itself.
* Scaling your query load means creating more blockchain endpoint nodes, which can be very expensive.

Demux solves these problems by off-loading queries to any persistence layer that you want. As blockchain events happen, your chosen persistence layer is updated by `updater` functions, which deterministically process an array of `Action` objects. The persistence layer can then be queried by your front-end through a suitable API (for example, REST or GraphQL).

This means that we can separate our concerns: for data that needs decentralized consensus of computation or access from other blockchain events, we can still store the data in indexed blockchain state, without having to worry about tailoring to front-end queries. For data required by our front-end, we can pre-process and index data in a way that makes it easy for it to be queried, in a horizontally scalable persistence layer of our choice. The end result is that both systems can serve their purpose more effectively.

### Side Effects

Since we have a system for acting upon specific blockchain events deterministically, we can utilize this system to manage non-deterministic events as well. These `effect` functions work almost exactly the same as `updater` functions, except they run asynchronously, are not run during replays, and modifying the deterministic datastore is off-limits. Examples include: signing and broadcasting a transaction, sending an email, and initiating a traditional fiat payment.

### Single Source of Truth

There are other solutions to the above problems that involve legacy persistence layers that are their own sources of truth. By deriving all state from the blockchain, however, we gain the following benefits:

* If the accumulated datastore is lost or deleted, it may be regenerated by replaying blockchain actions
* As long as application code is open source, and the blockchain is public, all application state can be audited
* No need to maintain multiple ways of updating state (submitting transactions is the sole way)

## Data Flow

<img src='https://i.imgur.com/MFfGOe3.png' height='492' alt='Demux Logo' />

1. Client sends transaction to blockchain
1. Action Watcher invokes Action Reader to check for new blocks
1. Action Reader sees transaction in new block, parses actions
1. Action Watcher sends actions to Action Handler
1. Action Handler processes actions through Updaters and Effects
1. Actions run their corresponding Updaters, updating the state of the Datastore
1. Actions run their corresponding Effects, triggering external events
1. Client queries API for updated data


## Class Implementations

Repository | Description
---|---
[EOSIO / demux-js-eos](https://github.com/EOSIO/demux-js-eos) * | Action Reader implementations for EOSIO blockchains
[EOSIO / demux-js-postgres](https://github.com/EOSIO/demux-js-postgres) * | Action Handler implementation for Postgres databases
[Zapata / demux-js-bitshares](https://github.com/Zapata/demux-js-bitshares) | Action Reader implementations for BitShares blockchain

*\* Officially supported by Block.one*

To get your project listed, add it here and submit a PR!


## Usage



This library provides the following classes:

* [**`AbstractActionReader`**](https://eosio.github.io/demux-js/classes/abstractactionreader.html): Abstract class used for implementing your own Action Readers

* [**`AbstractActionHandler`**](https://eosio.github.io/demux-js/classes/abstractactionhandler.html): Abstract class used for implementing your own Action Handlers   

* [**`BaseActionWatcher`**](https://eosio.github.io/demux-js/classes/baseactionwatcher.html): Base class that implements a ready-to-use Action Watcher

* [**`ExpressActionWatcher`**](https://eosio.github.io/demux-js/classes/expressactionwatcher.html): Exposes the API methods from the BaseActionWatcher through an Express server

In order to process actions, we need the following things:

- An implementation of an `AbstractActionReader`
- An implementation of an `AbstractActionHandler`
- At least one `HandlerVersion`, which contain `Updater` and `Effect` arrays

After we have these things, we need to:

- Instantiate the implemented `AbstractActionReader` with any needed configuration
- Instantiate the implemented `AbstractActionHandler`, passing in the `HandlerVersion` and any other needed configuration
- Instantiate the `BaseActionWatcher` (or a subclass), passing in the Action Handler and Action Watcher instances
- Start indexing via the Action Watcher's `watch()` method (by either calling it directly or otherwise)


#### Example

```javascript
const { BaseActionWatcher, ExpressActionWatcher } = require("demux")
const { MyActionReader } = require("./MyActionReader")
const { MyActionHandler } = require("./MyActionHandler")
const { handlerVersions } = require("./handlerVersions")
const { readerConfig, handlerConfig, pollInterval, portNumber } = require("./config")

const actionReader = new MyActionReader(readerConfig)
const actioHandler = new MyActionHandler(handlerVersions, handlerConfig)
```
Then, either
```javascript
const watcher = new BaseActionWatcher(
  actionReader,
  actionHandler,
  pollInterval,
)

watcher.watch()
```
Or,
```javascript
const expressWatcher = new ExpressActionWatcher(
  actionReader,
  actionHandler,
  pollInterval,
  portNumber,
)

expressWatcher.listen()

// You can then make a POST request to `/start` on your configured endpoint
```

### [**API documentation**](https://eosio.github.io/demux-js/)

### [Learn from a full example](examples/eos-transfers)
