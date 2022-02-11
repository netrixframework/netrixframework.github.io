---
layout: default
title: Writing tests
parent: Unit Tests
nav_order: 2
---

# Unit Tests

A unit test is represented by a `TestCase` object. Instantiate a `TestCase` object by specifying a `Name`, `Timeout`, `StateMachine` and a `FilterSet`

- `Name` is unit for a particular test scenario
- `Timeout` is the maximum duration after which the test is terminated
- `StateMachine` to assert a property
- `FilterSet` implements an interface for handling each event.

Additionally a `SetupFunc` can be specified which will can make initialization calls to the replicas. This can be used to prep the replicas for the test scenario being executed.

When running a unit test, **Netrix** waits for replicas to be ready and then invokes the filters for every event received from the replicas. Each invocation of `FilterSet` returns a list of `Message`s which are delivered to the intended replica. Refer [Data structures](https://pkg.go.dev/github.com/ds-test-framework/scheduler@v1.9.4/types) for documentation about data structure definitions used in this document.

## Context

Filters are invoked with an `Event` and a `Context` object which is meant for Filters to share information between successive calls. `Context` contains references to the `MessagePool`, `ReplicaStore`, `EventDAG` and `VarSet` which can be used when handling events.

- `MessagePool` contains a reference to the pool of messages received by **Netrix** indexed by Message ID
- `ReplicaStore` contains information about the replicas that are connected to **Netrix**. This can be used to fetch private key information when the filter intends to change a message.
- `EventDAG` contains a history of all the events that have been encountered as a Directed acyclic graph
- `VarSet` is a key value store that can be used to store information between success calls to the filters.

## FilterSet

`FilterSet` allows you to specify a sequence of `FilterFunc`s to handle events. 

```
type FilterFunc func(*Event, *Context) ([]*Message, bool)
```

The boolean value should be `true` if the `FilterFunc` handled the `Event` and `false` otherwise. `FilterSet` exposes `AddFilter(FilterFunc)` which should be used to add additional filters.

The filter set works as follows. For each event, the sequence of `FilterFunc`s is called until it returns `true` in the second return value. There is a default `FilterFunc` that handles events of type `MessageSend` and returns the corresponding `Message` referenced by the event. This means that when none of the `FilterFunc`s return `true` the message is delivered. If you use `FilterSet` without specifying any filters, then all messages are delivered by default.

Use `WithDefault` option when creating a `FilterSet` instance to specify a different default filters.

## Conditions and Actions

`FilterFunc` represents a generic function that can be used to write restrictions on messages. We provide a default `If().Then()`, `FilterFunc` that is relevant in most circumstances. The syntax is as follows,

```
filters.AddFilter(
    If(cond1).Then(action1, action2, ...)
)
filters.AddFilter(
    If(cond2).Then(action3, action2, ...)
)
```

`If` requires a `Condition` as an argument and `Then` accepts a list of `Action`s as parameters. Conditions are functions that return true/false based on the current event and context and Actions determine which messages to deliver. We provide a default set of conditions, actions and functions to compose Conditions. Refer [Conditions and Actions]({% link docs/tests/ca.md %}).

## State machines

State machines are used to specify properties that assert the success of the unit test. `testlib` allows you to specify deterministic state machines where transitions are labelled with `Condition` functions. State machines should be constructed using the builder interface. An example syntax,

```
sm := NewStateMachine()
initial := sm.Builder()
state2 := initial.On(cond1, "state1").On(cond2, "state2")
state2.On(cond3, SuccessState)
state2.On(cond4, FailureState)
```

The state machine takes a step for every event starting from the initial state, similar to the filter set. If any of the transition conditions returns true for the current event, the state machine transitions to that state. Any state can be marked as a success state,

```
state1.MarkSuccess()
```

## Test success/failure

**Netrix** continues executing the filter set and state machine until,

1. State machine transitions to `FailureState`
2. Timeout is triggered.

The `TestCase` succeeds if the state machine is in a success state at the end of the execution.

---

Next: [Conditions and Actions]({% link docs/tests/ca.md %})
