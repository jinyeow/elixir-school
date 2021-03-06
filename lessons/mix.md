# Mix

Before we can dive into the deeper waters of Elixir we first need to learn about mix. If you're familiar with Ruby mix is Bundler, RubyGems, and Rake combined. It's a crucial part of any Elixir project and in this lesson we're going to explore just a few of it's great features. To see all that mix has to offer run `mix help`.

Until now we've been working exclusively within `iex` which has limitations.  In order to build something substantial we need to divide our code up into many files to effectively manage it, mix let's us do that with projects.

From this point on we will focus on incrementally building a complete application. For our application, we'll build an HTTP service that will allow us to store some data and retrieve it later at a unique URL.

_Note_: Code for the application can be found at [doomspork/concoction](http://github.com/doomspork/concoction).

## Table of Contents

- [New Projects](#new-project)
- [Compilation](#compilation)
- [Interactive](#interactive)
- [Manage Dependencies](#manage-dependencies)
- [Environments](#environments)

## New Projects

When we're ready to create a new Elixir project, mix makes it easy with the `mix new` command.  This will generate our project's folder structure and necessary boilerplate.  This is pretty straight forward, so let's get started:

```bash
$ mix new concoction
```

From the output we can see that mix has created our directory and a number of boilerplate files:

```bash
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/concoction.ex
* creating test
* creating test/test_helper.exs
* creating test/concoction_test.exs
```

In this lesson we're going to focus our attention on `mix.exs`.  Here we configure our application, dependencies, environment, and version.  Open the file in your favorite editor, you should see something like this (comments removed for brevity):

```elixir
defmodule Concoction.Mixfile do
  use Mix.Project

  def project do
    [app: :concoction,
     version: "0.0.1",
     elixir: "~> 1.0",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  def application do
    [applications: [:logger]]
  end

  defp deps do
    []
  end
end
```

The first section we'll look at is `project`.  Here we define the name of our application (`app`), specify our version (`version`), Elixir version (`elixir`), and finally our dependencies (`deps`).

The `application` section is used during the generation of our application file which we'll cover next.

## Compilation

Mix is a smart and will compile your changes when necessary, but it may still be necessary explicitly compile your project.  In this section we'll cover how to compile our project and what compilation does.

To compile a mix project we only need to run `mix compile` in our base directory:

```bash
$ mix compile
```

There isn't much to our project so the output isn't too exciting but it should complete successfully:

```bash
Compiled lib/concoction.ex
Generated concoction app
```

When we compile a project mix creates a `_build` directory for our artifacts.  If we look inside `_build` we will see our compiled application: `concoction.app`.

## Interactive

It may be necessary to use `iex` within the context of our application.  Thankfully for us, mix makes this easy.  With our application compiled we can start a new `iex` session:

```bash
$ iex -S mix
```

Starting `iex` this way will loads your application and dependencies into the current runtime.

## Manage Dependencies

Our project doesn't have any dependencies but will shortly, so we'll go ahead and cover defining dependencies and fetching them.

To add a new dependency we need to first add it to our `mix.exs` in the `deps` section.  Our dependency list is comprised of tuples with two required values and one optional: The package name as an atom, the version string, and optional options.

For this example let's look at a project with dependencies, like [phoenix_slim](https://github.com/doomspork/phoenix_slim):

```elixir
def deps do
  [{:phoenix, "~> 0.16"},
   {:phoenix_html, "~> 2.1"},
   {:cowboy, "~> 1.0", only: [:dev, :test]},
   {:slim_fast, ">= 0.6.0"}]
end
```

As you probably decerned from the dependencies above, the `cowboy` dependency is only necessary during development and test.

Once we've defined our dependencies there is one final step, fetching them.  This is analogous to `bundle install`:

```bash
$ mix deps.get
```

That's it!  We've defined and fetched our project dependencies.  Now we're prepared to add dependencies to [concoction](https://github.com/doomspork/concoction) when the time comes.

## Environments

Mix, much like Bundler, supports differing environments.  Out of the box mix works with three environments:

+ `:dev` — The default environment.
+ `:test` — Used by `mix test`. Covered further in our next lesson.
+ `:prod` — Used when we ship our application to production.

The current environment can be accessed using `Mix.env`.  As expected, the environment can be changed via the `MIX_ENV` environment variable:

```bash
$ MIX_ENV=prod mix compile
```
