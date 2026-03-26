# conductor

Source Academy standard communication interface for languages

## Terminology

- Host: An environment from which Runners may be created, e.g. browser, CLI.
- Runner: An environment where user code is run.
- Evaluator: A program that processes user code and produces the result(s).
- Channel: A named bidirectional data stream between a Host and a Runner.
- Chunk: A piece of user code that is not associated with any file.
  This is usually strings from REPL, but contents of files can also be treated as chunks if there is no distinction to be made.
- Plugin: A program that provides additional functionality, loaded on demand;
  it may receive communications and communicate on declared Channels.

## Quick Start Guide
To run an evaluator using conductor using the frontend UI, follow these steps.

If your evaluator has been deployed to the [`language-directory`](https://github.com/source-academy/language-directory):
1. In the top right dropdown of the frontend UI, click Settings > Feature Flags
2. Enable the conductor.enable feature flag
3. If your language has been properly deployed, it will appear in the language dropdown in the top right
4. Select your language. It will now be run using conductor.

If your evaluator has not been deployed to the [`language-directory`](https://github.com/source-academy/language-directory):
1. In `src/commons/featureFlags/publicFlags.ts` the frontend repository, add these lines:
   - At the top of the file, add `import { flagConductorEvaluatorUrl } from 'src/features/conductor/flagConductorEvaluatorUrl';`
   - in the publicFlags array, add `flagConductorEvaluatorUrl`
2. In the top right dropdown of the frontend UI, click Settings > Feature Flags
3. Enable the conductor.enable feature flag
4. If your evaluator is hosted remotely, in the conductor.evaluator.url flag, input your evaluator's URL
5. If your evaluator is hosted locally, place your evaluator in `public/evaluators`. in the conductor.evaluator.url flag, input `/evlauator/YOUR_EVALUATOR.js`
6. The front end REPL will now run your evaluator using conductor


**For frontend developers:**
Load conductor from the GitHub repository rather than from npm. The GitHub source is the intended distribution source for this project:
```json
  "dependencies": {
    ...
    "conductor": "https://github.com/source-academy/conductor.git#0.3.0",
     ...
  }, ...
```

## Implementing a new language

To get started, use the `conductor-runner-example` template. It contains a basic evaluator that calls Javascript's `eval` on the provided chunks.

### The IEvaluator interface

To implement a new language using Conductor, implement the `conductor/runner/types/IEvaluator` interface.
This allows Conductor to interact with the language's implementation in a standard manner.

The `conductor/runner/BasicEvaluator` abstract class provides a basic implementation of this interface;
to use, implement `evaluateChunk` and override `evaluateFile` if needed.

### The entry point

An entry point should be created; this is the file initially executed to start a Runner.
It should construct an instance of `Conduit`, and register the `RunnerPlugin` with an argument of the evaluator.
`conductor/runner/util/initialise` can help with this (do `initialise(MyEvaluator)`).

Your implementation should be bundled using this file as the bundler's entry point.

### Language data

This is used by the host to locate your runner and execute it.
It should contain things like the path to your entry point, editor information, and other information about your language.
Consult [`language-directory` repository](https://github.com/source-academy/language-directory) for more information.

### Sending messages
To send messages between the runner and host, use these methods:

- `sendResult(result)` — sends evaluation results over the `__result` channel.
- `sendError(error)` — sends errors over the `__error` channel.
- `sendOutput(message)` — sends standard output/log messages over the `__stdio` channel.

These methods are available on the relevant runner plugin classes. Call the messaging methods on the plugin instance you receive in your evaluator or plugin code.

## Module interface

### Data types

Several standard data types are available for module-language interfacing.
Some are passed directly as JS values, others as identifiers. See `conductor/types/DataType`.
Note that all identifiers across data types must be unique - in other words, a raw identifier must be able to be mapped back to the correct data.

| Data type    | Passed as  | Notes                                                         |
| ------------ | ---------- | ------------------------------------------------------------- |
| void         | none\*     | The return type for functions with no return value            |
| boolean      | JS value   |                                                               |
| number       | JS value   | Per JS limitations, this is IEEE754 binary64 (`double` in C)  |
| const string | JS value   | Strings are immutable                                         |
| empty list   | none\*     | The empty-list value                                          |
| pair         | Identifier |                                                               |
| array        | Identifier | Arrays are singly-typed                                       |
| closure      | Identifier | Closures have fixed arity                                     |
| opaque       | Identifier | For values that can manipulated only by modules (e.g. a Rune) |
| list         | see notes  | Either a Pair (passed by identifier) or empty list (null\*)   |

\* as a convention, `undefined` is passed as the JS value for void type, and `null` is passed as the JS value for empty list type,
though it is always better to check the data type to be retrieved using the `*_type` functions than to test for equality using the JS values.
Currently, an equality comparison with null is required to distinguish between a `Pair` and the empty list when accepting a parameter of the type List,
though it is hoped that an alternative can be found soon.

Note that language implementations **are expected to verify that the types of arguments to external closures are correct**,
as it is not possible for the data type to be retrieved from a raw Identifier.

### Communication interface

In order to be language and evaluator-agnostic, modules will make no assumptions about the memory model of evaluators.
Thus, evaluators are responsible for providing functions to allow modules to read, manipulate, and create data.

Each of the data types passed as identifier have functions to create an instance of that data type,
as well as read and write data to it (or call it, in the case of closures). See `conductor/types/IDataHandler`.

### Standard library

A standard library of functions must be made available to modules. See `conductor/stdlib` for sample implementations.

## Plugins

Plugins provide additional functionality not provided by the base Conductor framework.

A Plugin is a class that implements `IPlugin` and contains the static field `channelAttach`: an array that specifies
the name(s) of the `Channel`(s) this plugin will be communicating on.

Upon registration of the Plugin with the Conduit, the class' constructor will be called with the following arguments:
- the instance of the `Conduit` to attach to
- an array of `Channel`s, corresponding to the entries in `channelAttach`, and
- any additional arguments passed to the `registerPlugin` method 

Most Plugins will not require the first argument, the `Conduit` instance. This is provided for the ability to
load other Plugins as needed, as well as to communicate between Plugins.

## Message Channels

The host (i.e., the frontend REPL) formats messages differently depending on the channel:
- Errors (from `__error`) appear in red. Used to send error messages.
- Output (from `__stdio`) appear in orange. Used to send IO messages (e.g. display calls).
- Results (from `__result`) appear in the default color. Used to display program return results.

If you are implementing your own `RunnerPlugin` (or similar plugin), you should use the `__result`, `__error`, and `__stdio` channels for results, errors, and output respectively to ensure compatibility with the host and consistent REPL formatting.
