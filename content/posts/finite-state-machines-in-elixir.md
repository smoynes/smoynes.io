Title: Modeling Finite State Machines in Elixir
Category: Elixir
Tags: elixir
Slug: fsm-in-elixir
Summary: Short version for index and feeds


A confession: I really love coffee but I don't like making it.

Coffee is a truly marvellous elixir composed of two basic ingredients:
water and the seed of a plant. A sip of coffee is a simple pleasure
that comes in many preparations: a simple product from a complicated
process.

To prepare a coffee is usually pretty complicated so I prefer to
purchase my coffee from a local cafe. It is great! They have a few
people behind the counter that make espresso and lattes and of course
someone to take my money.

While I drank my coffee at the local cafe one day I watched people as
they ordered their coffees and snacks. The cashier would tell the
barista the coffees to pull or the milks to steam, the kitchen the
sandwiches to prepare or the scones to plate. Sometimes the
conversation with the customer was garbled by the din of the cafe and
had to be repeated. Occasionally, a customer wouldn't know what they
wanted and the cashier waited.

I noticed this silly little dance, ordering coffee, was a bit like a
small distributed computation: each of actors in the cafe was a state
machine and they communicated through messages. I wondered if I could
model this system in a computer program.

I wondered if I could implement a model of this system in a
programming language that I am learning called Elixir.

(The post assumes a basic familiarity with Elixir syntax. I'll try
explain some of it, but the unfamiliar reader is encouraged to read
the [the crash course](http://elixir-lang.org/crash-course.html); it
should cover everything needed to follow along.)

Let's start small: with a single test. I want to test two simple behaviors:

- I can create and start a coffee purchaser state machine. When I
  create the state machine I want to initialize it with the guests
  order.
- When the state machine start, it begins in the `waiting` state.

<!-- -->

    :::elixir
    defmodule FsmExperimentsTest do
      use ExUnit.Case

      alias FsmExperiments.CoffeePurchaser

      setup do
        {:ok, %{order: [:espresso]}}
      end

      test "initial state is :waiting", %{order: order} do
        assert {:ok, fsm} = CoffeePurchaser.start_link order
        assert :waiting = CoffeePurchaser.current_state fsm
      end

    end

So what does this say? Not much, really. Just that when I create a new
coffee purchaser, it starts in the waiting state. The test calls
function called `start_link` that creates a new purchaser FSM and
passes it a list of things to buy; in this case, just one item: a
double espresso.

I also add a convenience for testing: a query event that asks the
state machine in which state it is currently. The test calls
`FsmExperiments.CurrentPurchaser.current_state` to get the current
state.

Now, to implement the purchaser state-machine using the `gen_fsm` OTP
behavior.

    ::::elixir
    defmodule FsmExperiments.CoffeePurchaser do

      def start_link(order) do
        :gen_fsm.start_link __MODULE__, order, []
      end

      def current_state(fsm) do
        :gen_fsm.sync_send_all_state_event fsm, :current_state
      end

      def init(_) do
        {:ok, :waiting, []}
      end

      def handle_sync_event(:current_state, _from, state, context) do
        {:reply, state, state, context}
      end
    end

To start a new state machine I create a new function that delegates to
the `gen_fsm.start_link` module passing three arguments: the name of
the module implementing the behavior, the initialization parameters,
and a list of options for creating the process (none, in this
case). The function will return a tuple, `{:ok, pid}`, to the caller
when the state machine is started and initialized.

The `gen_fsm` module will take care of creating a process for us and
calling the `init(order)` callback function within the process and
waiting for it to finish. The initialization function returns a tuple
to indicate success, `{:ok, initial_state, state_machine_context}`

Next, I want to implement the `current_state` query. But before I
that, a little background for people new to Elixir: the engine of
application state is the process. In object-oriented languages,
objects are responsible for storing state; in elixir we store state in
processes. << really bad >>

To get the current state of the purchaser, one needs to communicate
with the FSM process with a message. We want our FSM to handle this
message no matter which state it is in and respond to the calling
process with a reply message. That seems like a lot but the `gen_fsm`
module has support for this messaging pattern using
`sync_send_all_state_event(fsm, event)`: it sends a *synchronous
event*, `event` that can be handled in every state to the process
`fsm`.

When the `gen_fsm` behavior handles accepting the event and running
the callback function `handle_sync_event(event, from, state, context)`
in our module and passing a few parameters:

- `event`: the event that was sent; by convention the event is usually
   an atom (`:query_state`) or a tuple (`{:event_name, event_data}`)
   if there is data to be sent with the event;
- `from`: the process that sent the message;
- `state`: the state the finite state machine is in; and
- `context`: the data used by the process; in order to avoid confusion
  we call it this instead of the more typical `state`.

We make use of pattern matching in the function definition of
`CoffePurchaser.handle_sync_event`. The handler is declared to accept
`:query_state` events and returns a tuple `{:reply, response,
next_state, next_context}` to reply with `response` to the calling
process and to continue the current process in another state,
`next_state` with new process data, `next_context`. For a query event,
we don't want to change the FSM's state or process data so we return
those unchanged.


- coffee purchasing

- gen_fsm
- otp_dsl
- elixir agents
