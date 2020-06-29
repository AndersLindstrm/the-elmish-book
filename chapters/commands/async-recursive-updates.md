# Asynchronous Recursive Updates

In the previous section, we talked about how the command `Cmd.ofMsg` made our update function implicitly recursive. We also saw how to eliminate the use of said command using plain old control flow. This is not to say that recursive state updates are "bad" or shouldn't be utilized, it is just helpful to identify the implicit recursion pattern and simplify the code when possible. That said, recursive state updates can be really interesting and actually quite fun when *combined* with asynchronous workflows.

Similar to recursive functions, asynchronous recursive workflows can either be running indefinitely or stop running upon satisfying a base condition. An indefinite workflow can be a simple timer application that shows the current time and keeps updating it every second. A workflow that stops executing upon reaching a base condition is for example when you have a countdown that starts at a number and decreases the number every second until that number has reached zero. In the context of the Elmish application, we want to re-render the UI after every "step" of the asynchronous workflow. Re-rendering the UI means that with every step, we have to dispatch a message into the Elmish dispatch loop. Let's see this concept in action.

### Simple Timer

A timer keeps track of the current time, updating it every second. It looks something like this:

<div style="width:100%">
  <div style="margin: 0 auto; width:75%;">
    <resolved-image source="/images/commands/async-timer.gif" />
  </div>
</div>

The data model is rather simple, consider the types:
```fsharp
type State = {
    CurrentTime: DateTime
}

type Msg =
    | Tick
```
Using these messages, every time `Tick` is dispatched, the current time is updated and after one second, dispatches itself again.
```fsharp
let update (msg: Msg) (state: State) =
    match msg with
    | Tick ->
        let nextState = { state with CurrentTime = DateTime.Now }
        let step = async {
          do! Async.Sleep 1000
          return Tick
        }

        nextState, Cmd.fromAsync step
```
This way you get an application that ticks indefinitely but how to kick-start the recursion, *something* has to trigger `Tick` at some point right? Since I want to start the asynchronous workflow as soon as the program starts, I want to dispatch the `Tick` message from my `init()` function:
```fsharp
let init() = { CurrentTime = DateTime.Now }, Cmd.ofMsg Tick
```
Notice the proper usage of `Cmd.ofMsg` as it is executed once when the application starts up and not from the `update` function. Once started, the ticking cannot be stopped as there is no base condition to stop the recursion. Finally the `render` function is implemented to format the time and show it on screen:
```fsharp
let formatTime (time: DateTime) =
    let padZero n = if n < 10 then sprintf "0%d" n else string n
    sprintf "%s:%s:%s" (padZero time.Hour) (padZero time.Minute) (padZero time.Second)

let render (state: State) (dispatch: Msg -> unit) =
    Html.div [
        prop.style [ style.padding 20 ]
        prop.children [
            Html.h1 (formatTime state.CurrentTime)
        ]
    ]
```
That's it! Let's now look at another example where the asynchronous workflow terminates when it satisfies a condition, a countdown application.

### Implementing Countdown

The following application starts counting from an initial number (5 in this case) and decrements 1 every second until it reaches the base condition which is zero:

<div style="width:100%">
  <div style="margin: 0 auto; width:75%;">
    <resolved-image source="/images/commands/countdown.gif" />
  </div>
</div>

The data model is very similar to that of the timer. We keep track of the current count and the finish number. As for the messages, there is the message `StartCountdown` to kick off the recursive workflow and the `Tick` message that updates the state every second and calls itself again until the `Count` has reached zero:
```fsharp
type State = {
  Count : int
}

type Msg =
  | StartCountdown
  | Tick
```
When the application starts up, we dispatch `StartCountdown` immediately:
```fsharp
let init() = { Count = 5 }, Cmd.ofMsg StartCountdown
```
Then within the `update` function, the messages are handled as follows:
```fsharp { highlight: [11, 12] }
let update (msg: Msg) (state: State) =
    match msg with
    | StartCountdown ->
        let step = async {
            do! Async.Sleep 1000
            return Tick
        }

        state, Cmd.fromAsync step

    | Tick when state.Count <= 0 ->
        state, Cmd.none

    | Tick ->
        let nextState = { state with Count = state.Count - 1 }
        let step = async {
            do! Async.Sleep 1000
            return Tick
        }

        nextState, Cmd.fromAsync step
```
Notice the highlighted lines, here is where the program decides whether it has satisfied the base condition upon which it terminates the recursion. If that is the case, the state is returned as is without executing any further commands.

### Alternatives To Recursion

You might have thought: "there is definitely an easier way to do this". At the end of the day, we are just dispatching a bunch of messages one after the other with a delay in between. The only restriction we have at hand is the fact that `Cmd.fromAsync` only dispatches a single message that results from the asynchronous expression `Async<'Msg>` but nothing is stopping us from implementing a command that for example dispatches a message indefinitely between a delay of sleep:
```fsharp
module Cmd =
  let indefinite (timeout: int) (msg: 'Msg) =
    let command (dispatch: 'Msg -> unit) : unit =
      let workflow = async {
        while true do
          do! Async.Sleep timeout
          dispatch msg
      }

      Async.StartImmediate workflow

    Cmd.ofSub command
```
Using this command, the timer program can be simplified into the following:
```fsharp {highlight: [1]}
let init() = { CurrentTime =  DateTime.Now }, Cmd.indefinite 1000 Tick

let update msg update =
  match msg with
  | Tick -> { CurrentTime = DateTime.Now }, Cmd.none
```
Notice how we kick off the indefinite ticking in `init()` so that the `update` function doesn't need to implement any recursive behavior.

Likewise for the recursion with a base condition, we can implement a command that keeps dispatching a message until a certain criteria, a base condition is satisfied:
```fsharp
module Cmd =
  let indefiniteUntil (condition: unit -> bool) timeout msg =
      let command (dispatch: 'Msg -> unit) : unit =
          let workflow = async {
            while condition() do
              do! Async.Sleep timeout
              dispatch msg
          }

          Async.StartImmediate workflow

      Cmd.ofSub command
```
Theoretically, this could work but there is a big problem: how should we define the `condition: unit -> bool` parameter? If we provide a predicate that depends on the state (i.e. stop when `state.Count = 10`) then we would need to issue this command every time the state is updated which in itself brings more problems because that will start a separate asynchronous workflow on every state update! This problem is closely related to Elmish subscription, especially the types of subscriptions that can be canceled. This topic will be explored in-depth in a later section.