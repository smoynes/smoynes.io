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

To prepare a coffee is usually pretty complicated. You need to find
good beans that are well roasted and freshly ground as well some hot
water. Then you have to filter or press or steep or whatever. All of
this is way too much effort for me so I prefer to purchase my coffee
from a local cafe. It is great! They have a few people behind the
counter that know how to do all this and can work a pretty impressive
looking machine to make really delicious espresso.

While I drank my coffee at the local cafe one day, I watched people as
they ordered their coffees and snacks. The cashier would tell the
barista the coffees to pull or the milks to steam, the kitchen the
sandwiches to prepare or the scones to plate. Sometimes the
conversation with the customer was garbled by the din of the cafe and
had to be repeated. Occasionally, a customer wouldn't know what they
wanted and the cashier waited.

I noticed this silly little dance, ordering coffee, was a bit like a
small distributed computation: each of actors in the cafe was a state
machine and they communicated through messages. I wondered if I could
model this system in a programming language that I am learning called
Elixir.

(The post assumes a basic familiarity with Elixir syntax and data
types. I'll try explain some of it, but the unfamiliar reader is
encouraged to read the
[the crash course](http://elixir-lang.org/crash-course.html); it
should cover everything needed to follow along.)

Starting out
------------

Elixir includes a tool called `mix` that is used to compile elixir
projects and run development tasks. We can use it to create a project
skeleton.

<!-- starting-out-1 -->

    % mix new cafe
    * creating README.md
    * creating .gitignore
    * creating mix.exs
    * creating config
    * creating config/config.exs
    * creating lib
    * creating lib/cafe.ex
    * creating test
    * creating test/test_helper.exs
    * creating test/cafe_test.exs
    
    Your mix project was created successfully.
    You can use mix to compile it, test it, and more:
    
        cd cafe
        mix test
    
    Run `mix help` for more commands.

The `mix` command creates a few files that are mostly empty but it
gives us a place to start. There is a stub for a module,
`lib/cafe.ex`, and a test stub, `test/cafe_test.exs`, and some other
files we won't worry about here.

Elixir also has a tool called `iex` that is an interactive environment
or REPL

    % iex -S mix
    Erlang/OTP 17 [erts-6.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]
    
    Interactive Elixir (1.0.2) - press Ctrl+C to exit (type h() ENTER for help)
    iex(1)> Process.list()
    [#PID<0.0.0>, #PID<0.3.0>, #PID<0.6.0>, #PID<0.7.0>, #PID<0.9.0>, #PID<0.10.0>,
     #PID<0.11.0>, #PID<0.12.0>, #PID<0.13.0>, #PID<0.14.0>, #PID<0.15.0>,
     #PID<0.16.0>, #PID<0.17.0>, #PID<0.18.0>, #PID<0.19.0>, #PID<0.20.0>,
     #PID<0.21.0>, #PID<0.22.0>, #PID<0.23.0>, #PID<0.24.0>, #PID<0.25.0>,
     #PID<0.26.0>, #PID<0.27.0>, #PID<0.28.0>, #PID<0.36.0>, #PID<0.37.0>,
     #PID<0.38.0>, #PID<0.39.0>, #PID<0.40.0>, #PID<0.56.0>, #PID<0.57.0>,
     #PID<0.58.0>, #PID<0.59.0>, #PID<0.60.0>, #PID<0.68.0>, #PID<0.69.0>,
     #PID<0.70.0>, #PID<0.71.0>, #PID<0.72.0>, #PID<0.73.0>, #PID<0.74.0>,
     #PID<0.75.0>, #PID<0.77.0>]
    iex(2)> r Cafe
    lib/cafe.ex:1: warning: redefining module Cafe
    {:reloaded, Cafe, [Cafe]}

In the above example, I call a function and can see the results as
well as recompile a module so that any changes to the source file are
included in the `iex` session. We'll be using these commands a lot to
explore our cafe system.

Agent zero
----------

There are a few options for how to model a state machine using
Elixir. They all have tradeoffs, like any design decision, but I'll
start with the simplest and see how far we can get with that.


- I can create a cafe customer state machine.
- When the state machine starts, it begins in the `waiting` state.
- For testing convenience, I want to be able to get the state
  machine's current state.

<!-- -->

    :::elixir
    defmodule Cafe.Customer do
    
      def start do
        Agent.start &init/0
      end
    
      def stop(fsm) do
        Agent.stop fsm
      end
    
      def init do
        %{state: :idle}
      end
    
      def current_state(fsm) do
        Agent.get fsm, fn ctx -> ctx.state end
      end
      
    end

<!-- -->

    iex(1)> {:ok, pid} = Cafe.Customer.start
    {:ok, #PID<0.88.0>}
    iex(2)> Cafe.Customer.current_state pid
    :idle
    iex(3)> Cafe.Customer.stop pid
    :ok
    iex(4)> Cafe.Customer.current_state pid
    ** (exit) exited in: GenServer.call(#PID<0.88.0>, {:get, #Function<1.60568772/1 in Cafe.Customer.current_state/1>}, 5000)
        ** (EXIT) no process
        (elixir) lib/gen_server.ex:356: GenServer.call/3


explain:

- defmodule
- use
- alias
- test context
- asserts

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
