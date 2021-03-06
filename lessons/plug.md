# Plug

We will continue our development of [Concoction](http://github.com/doomspork/concoction) in this lesson by learning about and integrating Plug.

If you're familiar with Ruby you can think of Plug as Rack with a splash of Sinatra, it provides a specification for web application components and adapters for web servers. While not part of Elixir core, Plug is an official Elixir project.

## Table of Contents

- [Installation](#installation)
- [The specification](#the-specification)
- [Creating a Plug](#creating-a-plug)
- [Using Plug.Router](#using-plugrouter)
- [Running our web app](#running-our-web-app)
- [Testing Plugs](#testing-plugs)
- [Available Plugs](#available-plugs)

## Installation

Installation is a breeze with mix.  To install Plug we need to make two small changes to our `mix.exs`.  The first thing to do is add both Plug and a web server to our file as dependencies, we'll be using Cowboy:

```elixir
defp deps do
  [{:cowboy, "~> 1.0.0"},
   {:plug, "~> 1.0"}]
end
```

The last thing we need to do is add both our web server and Plug to our OTP application:

```elixir
def application do
  [applications: [:cowboy, :logger, :plug]]
end
```

## The specification

In order to begin creating Plugs we need to know, and adhere to, the Plug spec.  Thankfully for us, there are only two functions necessary: `init/1` and `call/2`.

The `init/1` function is used to initialize our Plug's options, passed as the second argument to our `call/2` function.  In addition to our initialized options the `call/2` function receives a `%Plug.Conn` as it's first argument and is expected to return a connection.

Here's a simple Plug that returns "Hello World!":

```elixir
defmodule HelloWorldPlug do
  import Plug.Conn

  def init(options), do: options

  def call(conn, _opts) do
    conn
    |> put_resp_content_type("text/plain")
    |> send_resp(200, "Hello World!")
  end
end
```

## Creating a Plug

For our project we're going to create a Plug to verify whether or not the request has some set of required parameters.  By impleneting our validation in a Plug we can be assured that only valid requests will make it through to our application.  We will expect our Plug to be initialized with two options: `:paths` and `:fields`.  These will represent the paths we apply our logic to and which fields to require.

_Note_: Plugs are applied to all requests which is why we will handle filtering requests and applying our logic to only a subset of them.  To ignore a request we simply pass the connection through.

We'll start by looking at our finished Plug and then discuss how it works, we'll create it at `lib/plug/verify_request.ex`:

```elixir
defmodule Concoction.Plug.VerifyRequest do
  import Plug.Conn

  defmodule IncompleteRequestError do
    @moduledoc """
    Error raised when a required field is missing.
    """

    defexception message: "", plug_status: 400
  end

  def init(options), do: options

  def call(%Plug.Conn{request_path: path} = conn, opts) do
    if path in opts[:paths], do: verify_request!(conn.body_params, opts[:fields])
    conn
  end

  defp verify_request!(body_params, fields) do
    verified = body_params
               |> Map.keys
               |> contains_fields?(fields)
    unless verified, do: raise IncompleteRequestError
  end

  defp contains_fields?(keys, fields), do: Enum.all?(fields, &(&1 in keys))
end
```

The first thing to note is we have defined a new exception `IncompleteRequestError` and that one of it's options is `:plug_status`.  When available this option is used by Plug to set the HTTP status code in the event of an exception.

The second portion of our Plug is the `call/2` method, this is where we handle whether or not to apply our verification logic.  Only when the request's path is contained in our `:paths` option will we call `verify_request!/2`.

The last portion of our plug is the private function `verify_request!/2` which verifies whether the required `:fields` are all present.  In the event that some are missing, we raise `IncompleteRequestError`.

## Using Plug.Router

Now that we have our `VerifyRequest` plug, we can move on to our router.  As we are about to see we don't need a framework like Sinatra in Elixir, we get that for free with Plug.

To start let's create a file at `lib/plug/router.ex` and copy the following into it:

```elixir
defmodule Concoction.Plug.Router do
  use Plug.Router

  plug :match
  plug :dispatch

  get "/", do: send_resp(conn, 200, "Welcome")
  match _, do: send_resp(conn, 404, "Opps!")
end
```

This is a bare minimum Router but the code should be pretty self explainatory.  We've included some macros through `use Plug.Router` and then set up two of the builtin Plugs: `:match` and `:dispatch`.   There are two defined routes, one for handling GET returns to the root and the second for matching all other requests so we can return a 404 message.

Let's add our Plug to the router:

```elixir
defmodule Concoction.Plug.Router do
  use Plug.Router

  alias Concoction.Plug.VerifyRequest

  plug Plug.Parsers, parsers: [:urlencoded, :multipart]
  plug VerifyRequest, fields: ["content", "mimetype"],
                      paths:  ["/upload"]
  plug :match
  plug :dispatch

  get "/", do: send_resp(conn, 200, "Welcome")
  post "/upload", do: send_resp(conn, 201, "Uploaded")
  match _, do: send_resp(conn, 404, "Opps!")
end
```

That's it!  We've setup our Plug to verify that all requests to `/upload` include both `"content"` and `"mimetype"`, only then will route code be executed.

For now our `/upload` endpoint isn't very useful but we've seen how to create and integrate our Plug. In the next lessons we'll add more functionality.

## Running our web app

Before we can run our application we need to setup and configure our web server, in this instance Cowboy.  For now we'll just make the code changes necessary to run everything, we'll dig into specifics in later lessons.

Let's start by updating the `application` portion of our `mix.exs` to tell Elixir about our application and set an application env variable.  With those changes in place our code should look something like this:

```elixir
def application do
  [applications: [:cowboy, :plug],
   mod: {Concoction, []},
   env: [cowboy_port: 8080]]
end
```

Next we need to update `lib/concoction.ex` to start and supervisor Cowboy:

```elixir
defmodule Concoction do
  use Application

  def start(_type, _args) do
    port = Application.get_env(:concoction, :cowboy_port, 8080)

    children = [
      Plug.Adapters.Cowboy.child_spec(:http, Concoction.Plug.Router, [], port: port)
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

Now to run our application we can use:

```shell
$ mix run --no-halt
```

## Testing a Plug

Testing Plugs is pretty straight forward thanks to `Plug.Test`, it includes a number of convience functions to make testing easy.

See if you can follow along with router test:

```elixir
defmodule RouterTest do
  use ExUnit.Case
  use Plug.Test

  alias Concoction.Plug.Router

  @content "<html><body>Hi!</body></html>"
  @mimetype "text/html"

  @opts Router.init([])

  test "returns welcome" do
    conn = conn(:get, "/", "")
           |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 200
  end

  test "returns uploaded" do
    conn = conn(:post, "/upload", "content=#{@content}&mimetype=#{@mimetype}")
           |> put_req_header("content-type", "application/x-www-form-urlencoded")
           |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 201
  end

  test "returns 404" do
    conn = conn(:get, "/missing", "")
           |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 404
  end
end
```

The remaining tests can be found in the [Concoction](https://github.com/doomspork/concoction) repo.

## Available Plugs

There are a number of Plugs available out-of-the-box, the complete list can be found in the Plug docs [here](https://github.com/elixir-lang/plug#available-plugs).
