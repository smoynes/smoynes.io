Title: Modeling Finite State Machines in Elixir
Category: Elixir
Tags: elixir
Slug: fsm-in-elixir
Summary: A look at how to create state machines using Agents

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
skeleton:

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

The `mix new` command creates a few files that are mostly empty but it
gives us a place to start. There is a stub for a module
(`lib/cafe.ex`), a test stub (`test/cafe_test.exs`), and some other
files we won't worry about here.

Elixir also has a tool called `iex` that is an interactive environment
or REPL:

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

There are a few options for how to implement a state machine using
Elixir. They each have tradeoffs, like any design decision, but I'll
start with the simplest and see how far we can get with that.

* explain processes and state *

The `Agent` module provides a very simple abstraction around state and
a simple API for querying and updating the state.

<!-- agent-zero-1 -->

    :::elixir
    lib/customer.ex

    defmodule Cafe.Customer do

      def start do
        Agent.start &init/0
      end

      def stop(fsm) do
        Agent.stop fsm
      end

      def current_state(fsm) do
        Agent.get fsm, &query_state/1
      end

      def init do
        %{state: :idle}
      end

      def query_state(ctx) do
        ctx.state
      end

    end

We can start and stop a finite state machine and query the current
state by delegating to the Agent module.

Let's try it out in the console:

<!-- -->

    iex(1)> {:ok, pid} = Cafe.Customer.start
    {:ok, #PID<0.88.0>}
    iex(2)> Cafe.Customer.current_state pid
    :idle
    iex(3)> Cafe.Customer.stop pid
    :ok

Well, that is pretty cool. With relatively few lines of code, I've got
a basic finite state machine with a single state.

// Introduce full state machine.

Let's try to add all the states and transitions for our state machine,
ignoring any actions the customer has to perform.

<!-- agent-zero-2 -->

    ::::elixir
    lib/customer.ex

    defmodule Cafe.Customer do

      def order(fsm, order) do
        Agent.update fsm, &handle_event(:order, &1, order)
      end

      def cashier(fsm, cashier_pid) do
        Agent.update fsm, &handle_event(:cashier, &1, cashier_pid)
      end

      def total(fsm, total) do
        Agent.update fsm, &handle_event(:total, &1, total)
      end

      def confirm(fsm, confirmation) do
        Agent.update fsm, &handle_event(:confirm, &1, confirm)
      end

      def handle_event(:order, %{state: :idle} = ctx, order) do
        Enum.into [state: :wait_cashier, order: order], ctx
      end

      def handle_event(:order, %{state: :wait_order} = ctx, order) do
        Enum.into [state: :paying, order: order], ctx
      end

      def handle_event(:cashier, %{state: :idle} = ctx, cashier) do
        Enum.into [state: :wait_order, cashier: cashier], ctx
      end

      def handle_event(:cashier, %{state: :wait_cashier} = ctx, cashier) do
        Enum.into [state: :paying, cashier: cashier], ctx
      end

      def handle_event(:total, %{state: :paying} = ctx, total) do
        Enum.into [state: :wait_delivery, total: total], ctx
      end

      def handle_event(:delivery, %{state: :wait_delivery} = ctx, confirmation) do
        Enum.into [state: :done, confirmation: confirmation], ctx
      end

    end

<!-- agent-zero-3 -->

    ::::elixir
    lib/customer.ex

    defmodule Cafe.Customer do

      def order(fsm, order) do
        send_event fsm, :order, order
      end

      def cashier(fsm, cashier_pid) do
        send_event fsm, :cashier, cashier_pid
      end

      def total(fsm, total) do
        send_event fsm, :total, total
      end

      def confirm(fsm, confirmation) do
        send_event fsm, :confirm, confirmation
      end

      def send_event(fsm, event, message) do
        Agent.update fsm, __MODULE__, :handle_event, [event, message]
      end

      def handle_event(ctx, event, message) do
        do_handle_event(ctx, event, message)
        |> Enum.into ctx
      end

      def do_handle_event(%{state: :idle}, :order, order) do
        [state: :wait_cashier, order: order]
      end

      def do_handle_event(%{state: :wait_order},:order, order) do
        [state: :paying, order: order]
      end

      def do_handle_event(%{state: :idle}, :cashier, cashier) do
        [state: :wait_order, cashier: cashier]
      end

      def do_handle_event(%{state: :wait_cashier}, :cashier, cashier) do
        [state: :paying, cashier: cashier]
      end

      def do_handle_event(%{state: :paying}, :total, total) do
        [state: :wait_delivery, total: total]
      end

      def do_handle_event(%{state: :wait_delivery}, :delivery, confirmation) do
        [state: :done, confirmation: confirmation]
      end

    end


- coffee purchasing
- gen_fsm
- otp_dsl
- elixir agents
