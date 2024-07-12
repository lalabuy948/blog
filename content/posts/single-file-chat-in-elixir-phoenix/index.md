---
title: "Single File Chat in Elixir Phoenix"
date: 2024-07-12T23:46:53+02:00
draft: false
comments: true
cover: "img/single-file-chat-in-elixir-phoenix/Hero.webp"
tags:
  - code
  - elixir
  - tutorial

seo: ["Elixir", "Phoenix", "LiveView", "Chat Application", "Real-time Messaging", "GenServer", "Phoenix.PubSub", "Web Development", "LiveBook", "Elixir Tutorial"]
seo_description: "Learn how to create a single-file chat application in Elixir Phoenix with real-time messaging using GenServer and Phoenix.PubSub. This tutorial walks you through setting up message storage, handling updates, and integrating LiveView for seamless user experiences. Perfect for developers looking to enhance their Elixir and Phoenix skills."
---

## Meet [PhonenixPlayground](https://hexdocs.pm/phoenix_playground/readme.html)

I found this library on github suggestions and I always wanted to have something simple and quick to test out LiveView prototypes. What can be quicker than opening Livebook and trying something there? Folks behind [PhonenixPlayground](https://hexdocs.pm/phoenix_playground/readme.html) made both of my wishes come true!

If you would like jump straight into the code:

[![Run in Livebook](https://livebook.dev/badge/v1/pink.svg)](https://livebook.dev/run?url=https%3A%2F%2Fmrpopov.com%2Fposts%2Fsingle-file-chat-in-elixir-phoenix%2Fphoenix-dirty-chat.livemd)

<!--more-->

## What's the plan?

- Have chat with exchanging messages between clients in real time
- We will use [Phoenix.LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html)
- We will have `MessageStorage` module which holds messages using [GenServer](https://hexdocs.pm/elixir/GenServer.html)
- We will delete old messages once in a while
- We will use [Phoenix.PubSub](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html) for updating clients via LiveView sockets
- Have all of it in a single LiveBook

## Let's jump into

Open empty livebook and add next block to Notebook Dependencies and setup

```elixir
Mix.install([
  {:phoenix_playground, "~> 0.1.3"}
])
```

## Let's create logic to hold our messages

First I created simple `MessageStorage` module with three key functions,

- `init` defines our initial state
- `add_message/1` I hope it's self explanatory
- `get_messages/1` likewise

Now we need to handle GenServer events:

`handle_cast/2` simmply add new message to the begning of our empty list which we initiated in `init/1`.

and

`handle_call` where I used `Enum.reverse()` in order to return messages in reverse, because in the chat you read top down where old on top and newest at the bottom.

```elixir
defmodule MessageStorage do
  use GenServer

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_args) do
    {:ok, []}
  end

  def add_message(message) do
    GenServer.cast(__MODULE__, {:add_message, message})
  end

  def get_messages() do
    GenServer.call(__MODULE__, :get_messages)
  end

  def handle_cast({:add_message, message}, state) do
    {:noreply, [message | state]}
  end

  def handle_call(:get_messages, _from, state) do
    {:reply, Enum.reverse(state), state}
  end
end
```

## Let's add clean up and updates

We will define clean up interval and schedule clean up job every 5 seconds to delete the oldest message.

> ⁉️ This is not optimal way to delete old messages, only for educational purposes, as proper way would be to store created time stamp next to each message and delete based on time stamps, but just for quick night dirty prototyping it will do the job.

```elixir
defmodule MessageStorage do
  use GenServer

  @cleanup_interval 5_000 # 5 seconds

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_args) do
    schedule_cleanup()
    {:ok, []}
  end

  ...

  def handle_info(:cleanup, state) do
    new_state = clean_oldest_message(state)
    schedule_cleanup()

    ...

    {:noreply, new_state}
  end

  defp schedule_cleanup do
    Process.send_after(self(), :cleanup, @cleanup_interval)
  end

  defp clean_oldest_message([]), do: []
  defp clean_oldest_message(state) do
    List.delete_at(state, -1)
  end
```

As for the updates as prommissed we will use [Phoenix.PubSub](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html) as we will need to tell to our clients what's going on like new message was added to the view and old ones were deleted.

To achieve that we will need two core functionalities: subscribe and broadcast.
`subscribe` will let our clients to subscribe to our events from `MessageStorage` module

and

`broadcast` will enable us to send new updates into phonix PubSub.

Just jumping a bit further, once we will add Phoenix.PubSub to our star tree we will define name of that PubSub like this:

```elixir
Supervisor.child_spec({Phoenix.PubSub, name: :demo_pub_sub}, id: :demo_pub_sub),
```

That's why here you can see that name:

```elixir
Phoenix.PubSub.subscribe(:demo_pub_sub, "message_topic")
```

and `message_topic` is just the name for our topic. Here is extended functionality of module:

```elixir
defmodule MessageStorage do
  use GenServer

  ...

  def handle_info(:cleanup, state) do
    new_state = clean_oldest_message(state)
    schedule_cleanup()

    Phoenix.PubSub.broadcast(:demo_pub_sub, "message_topic", {:message_last_deleted})

    {:noreply, new_state}
  end

  def subscribe, do: Phoenix.PubSub.subscribe(:demo_pub_sub, "message_topic")

  defp broadcast({:error, _reason} = error, _event), do: error
  defp broadcast({:ok, message}, event) do
    Phoenix.PubSub.broadcast(:demo_pub_sub, "message_topic", {event, message})
    {:ok, message}
  end

  defp schedule_cleanup do
    Process.send_after(self(), :cleanup, @cleanup_interval)
  end

  defp clean_oldest_message([]), do: []
  defp clean_oldest_message(state) do
    List.delete_at(state, -1)
  end
end
```

### What it does?

- Once client subscribed to our topic it will able to react to updates.
- Once cleanup happened we send update to the Phoenix.PubSub saying message deleted.
- Once new message added we tell to everyone who subscribed that new message was added.

## Let's move to our LiveView

We need to create another module which will hold Phoenix.LiveView logic.

Quite similiar to what we've done in `MessageStorage` we need to init the state. And subscribe to our message topic from `Phoenix.PubSub`.

And as you can see we have `socket` through which we able to send and receive updates from our page. At the return we will iniate `messsages` state and `message_value` which we will use for our input value.

```elixir
defmodule DemoTest do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    if connected?(socket), do: MessageStorage.subscribe()
    messages = MessageStorage.get_messages()

    {:ok, assign(socket, messages: messages, message_value: "")}
  end

  ...

end
```

> ❤️ Huge thanks to the everyone who worked on erlang, BEAM, Elixir, Phoenix and LiveView that I don't need to write a single line of javascript. (no offend, I'm a backend developer and you will notice it from my frontend skills)

In our render function we can write html and `heex` annotaions.
I simply added two css libraries as otherwise it would look completelly boring and aweful, which can be found in the `<head></head>`.

Let's display our messages, for that we will use `for` anotation:
`<div class="chat-bubble" :for={message <- @messages}><%= message %></div>`

And in order to "submit" our message we will use simple form:

```html
<form phx-submit="new_message" >
  <input type="text" name="text" value={@message_value} placeholder="Message..." autofocus>
  <input type="submit" />
</form>
```

In which you can find our `@message_value` which we added in init state and `phx-submit="new_message"`, this is an event which we will need to handle in order to send updates from our UI to `MessageStorage` and than further to other clients.

```elixir
defmodule DemoTest do
  use Phoenix.LiveView

  ...

  def render(assigns) do
    ~H"""
    <head>
      <link href="https://cdn.jsdelivr.net/npm/daisyui@4.12.10/dist/full.min.css" rel="stylesheet" type="text/css" />
      <script src="https://cdn.tailwindcss.com"></script>
    </head>

    <body>

      <div class="h-screen flex items-center justify-center">

        <div class="mockup-phone border-primary">
          <div class="camera"></div>
          <div class="display">
            <div class="artboard artboard-demo phone-1">

              <div class="chat chat-end w-full">
                <div class="chat-bubble" :for={message <- @messages}><%= message %></div>
              </div>

              <form phx-submit="new_message" >
                <input type="text" name="text" value={@message_value} placeholder="Message..." autofocus>
                <input type="submit" />
              </form>

            </div>
          </div>
        </div>

      </div>

    </body>
    """
  end

  ...

end
```

Now let's wire our UI logic with the rest of the funcitonality which we developed.

### What needs to be done?

- Handle `"new_message"`:
  - send new message to `MessageStorage` in order to store it in `GenServer` and update other clients via `Phoenix.PubSub`
  - and we need to update socket state in order to display it here in our LiveView: `<div class="chat-bubble" :for={message <- @messages}><%= message %></div>`
- Handle updates from `GenServer`
  - `:message_last_deleted` we need to delete last one from our view
  - `:message_created` display new message which came from GenServer which was sent from another client. (or by you from iex)


```elixir
defmodule DemoTest do
  use Phoenix.LiveView

  ...

  def handle_event("new_message", %{"text" => message}, socket) do
    MessageStorage.add_message(message)

    # here i wanted to reset message once we hit enter, but found out it's not a bug but feature
    # works only once (input is in focus and due to that it wont be updated)
    # https://github.com/phoenixframework/phoenix_live_view/issues/624

    {:noreply, socket}
  end

  def handle_info({:message_last_deleted}, socket) do
    if length(socket.assigns.messages) == 0 do
      {:noreply, socket}
    else
      [_|tail] = socket.assigns.messages
      {:noreply, assign(socket, messages: tail )}
    end
  end

  def handle_info({:message_created, message}, socket) do
    {:noreply, assign(socket, messages: socket.assigns.messages ++ [message] )}
  end
end
```

## Let's run

We need to start our `Phoenix.PubSub`, `GenServer` which we wrapped in `MessageStorage` module and our `Phoenix.LiveView` via `PhoenixPlayground.start/2`

```elixir
children = [
  Supervisor.child_spec({Phoenix.PubSub, name: :demo_pub_sub}, id: :demo_pub_sub),
  MessageStorage,
]

PhoenixPlayground.start(live: DemoTest, child_specs: children)
```

You can run it and test yourself:

[![Run in Livebook](https://livebook.dev/badge/v1/blue.svg)](https://livebook.dev/run?url=https%3A%2F%2Fmrpopov.com%2Fposts%2Fsingle-file-chat-in-elixir-phoenix%2Fphoenix-dirty-chat.livemd)
