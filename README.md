# Zeebe Node.js Client

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![CircleCI](https://circleci.com/gh/CreditSenseAU/zeebe-client-node-js/tree/master.svg?style=svg)](https://circleci.com/gh/CreditSenseAU/zeebe-client-node-js/tree/master)

This is a Node.js gRPC client for [Zeebe](https://zeebe.io). It is written in TypeScript and transpiled to JavaScript in the `dist` directory.

Comprehensive API documentation is available [online](https://creditsenseau.github.io/zeebe-client-node-js/) and in the `docs` subdirectory.

## Example Use

### Add the Library to your Project

```bash
npm i zeebe-node
```

### Get Broker Topology and List Workflows

```javascript

 const ZB = require('zeebe-node');

(async() => {
    const zbc = new ZB.ZBClient("localhost:26500");
    const topology = await zbc.topology();
    console.log(JSON.stringify(topology, null, 2));

    let workflows = await zbc.listWorkflows();
    console.log(workflows);
})();

```

### Deploy a workflow

```javascript

 const ZB = require('zeebe-node');

(async() => {
    const zbc = new ZB.ZBClient("localhost:26500");

    const res = await zbc.deployWorkflow("./domain-mutation.bpmn");

    console.log(res);
})();

```

### Create a Task Worker

```javascript

const ZB = require('zeebe-node');

(async() => {
    const zbc = new ZB.ZBClient("localhost:26500");

    const zbWorker = zbc.createWorker("test-worker", "demo-service", handler);
})();

function handler(job, complete){
    console.log("Task payload", job.payload);
    let updatedPayload = Object.assign({}, job.payload, {updatedProperty: "newValue"});

    // Task worker business logic goes here

    complete(updatedPayload);
}

```

Here is an example payload:

```javascript

{ key: '578',
  type: 'demo-service',
  jobHeaders:
   { workflowInstanceKey: '574',
     bpmnProcessId: 'test-process',
     workflowDefinitionVersion: 1,
     workflowKey: '3',
     elementId: 'ServiceTask_0xdwuw7',
     elementInstanceKey: '577' },
  customHeaders: '{}',
  worker: 'test-worker',
  retries: 3,
  deadline: '1546915422636',
  payload: { testData: 'something' } }

```

The worker can be configured with options. Shown below are the defaults that apply if you don't supply them:

```javascript

const workerOptions = {
    maxActiveJobs: 32, // the number of simultaneous tasks this worker can handle
    timeout: 1000, // the maximum amount of time the broker should allow this worker to complete a task
}

const onConnectionError = (err) => console.log(err) // Called when the connection to the broker cannot be established, or fails

const zbWorker = zbc.createWorker("test-worker", "demo-service", handler, workerOptions, onConnectionError);

```

### Start a Workflow Instance

```javascript

const ZB = require("zeebe-node");

(async() => {
    const zbc = new ZB.ZBClient("localhost:26500");
    const result = await zbc.createWorkflowInstance("test-process", {testData: "something"});
    console.log(result);
})();

```

Example output:

```javascript

{ workflowKey: '3',
  bpmnProcessId: 'test-process',
  version: 1,
  workflowInstanceKey: '569' }

```

### Publish a Message

```javascript

    const zbc = new ZB.ZBClient("localhost:26500");
    zbc.publishMessage({
            correlationKey: "value-to-correlate-with-workflow-payload",
            messageId: uuid.v4(),
            name: "message-name",
            payload: {valueToAddToWorkflowPayload: "here", status: "PROCESSED"},
            timeToLive: 10000
        });

```

You can also publish a message targeting a [Message Start Event](https://github.com/zeebe-io/zeebe/issues/1858).
In this case, there is no correlation key, and all Message Start events that match the `name` property will receive the message.

You can use the `publishStartMessage()` method to publish a message with no correlation key (it will be set to `__MESSAGE_START_EVENT__` in the background):

```javascript

    const zbc = new ZB.ZBClient("localhost:26500");
    zbc.publishStartMessage({
            messageId: uuid.v4(),
            name: "message-name",
            payload: {initialWorkflowPayloadValue: "here"},
            timeToLive: 10000
        });

```

### Graceful Shutdown

To drain workers, call the `close()` method of the ZBClient. This causes all workers using that client to stop polling for jobs, and returns a Promise that resolves when all active jobs have either finished or timed out.

```javascript

    console.log('Closing client...');
    zbc.close().then(() => console.log('All workers closed'));

```

### Generating TypeScript constants for BPMN Processes

Message names and Task Types are untyped magic strings. The `BpmnParser` class provides a static method `generateConstantsForBpmnFiles()`.
This method takes a filepath and returns TypeScript definitions that you can use to avoid typos in your code, and to reason about the completeness of your task worker coverage.

```javascript

const ZB = require('zeebe-node');
(async () => {
    console.log(await ZB.BpmnParser.generateConstantsForBpmnFiles(workflowFile));
})();

```

This will produce output similar to:

```typescript
// Autogenerated constants for msg-start.bpmn

export enum TaskType = {

    CONSOLE_LOG = "console-log"

};

export enum MessageName = {

    MSG_EMIT_FRAME = "MSG-EMIT_FRAME",
    MSG_START_JOB = "MSG-START_JOB"

};

```

## Debugging

Enable debug output for the client library using the [debug](https://www.npmjs.com/package/debug) module by setting the environnment variable `DEBUG` to include `zeebe-node:*`.

For example:

```
DEBUG=zeebe-node:* node app.js
```

## Developing

The source is written in TypeScript in `src`, and compiled to ES6 in the `dist` directory.

To build:

```bash
npm run build
```

To start a watcher to build the source and API docs while you are developing:

```bash
npm run dev
```

### Tests

Tests are written in Jest, and live in the `src/__tests__` directory. To run the tests:

```bash
npm t
```
