# EOS (Explore OS)

This project is an experimental **actor system** thats intended to be highly interactive.
The goal is to provide an environment that makes it easy to prototype different scenarios
and explore the behavior of systems.

- [Work in Progress](#work-in-progress)
- [Introduction](#introduction)
- [Why?](#why)
- [Getting started](#getting-started)
- [Workspace](#workspace)
- [Teleplot](#teleplot)
- [How it works](#how-it-works)

---

## Work in Progress
Before we get into any details, please not that everything here is still pretty much `Work in Progress`,
so do expect things to break. Once the project is stable enough, I will remove this notice.

## Introduction
This project is an experiment in building an **actor system** that can be viewed and modified at runtime.

While it’s inspired by the actor model, you don’t need prior actor experience to understand the idea:

- You can think of each **actor** as a service.
- Sending a **message** is like calling a function or making a request to a server.
- Whether you picture it as microservices, a monolith, or distributed servers, the same model applies.

The goal is to create a **lightweight playground** where you can:

- **Prototype microservice interactions**: watch requests and responses flow between simplified services.
- **Experiment with execution order**: test “what happens if A runs before B” in a game or system.
- **Inspect and modify at runtime** (planned): pause the system, edit messages, and resume execution.

Think of it as turning a container into a **navigable debugger for your whole system**.
It’s not production-ready and it’s slower than real-world setups—but that’s the point: it’s a safe,
transparent environment to explore how complex systems behave.

---

## Why?
I was thinking about how to design a VM, for an actor based language, thats image based like smalltalk,
while still making it possible to use vscode (or any other editor for that matter) for editing the source code.
My first idea was to create a user mode file system with fuse that could map the code to files
in order for the editor to pick it up, but then I realized, that you could represent more than just source code.
For instance the internal state of actors. Then it dawned on me that you could also map all the messages in the
system like that and basically make everything explorable using any editor or basic unix tools like cd, ls and cat.

The only problem was, I have never made a file system with fuse and by the looks of it, it seems rather complicated.
So I decided to build a prototype that doesn't map the internal state of a VM onto a file system, but use the
file system to store its internal state instead.

You're probably already thinking of around a hundred reasons why this is a bad idea and I just want to say
that I agree. Using this in a production system would bring all sorts of problems with it.
It's inefficient, unsecure and probably many other things I can't think of right now.

So why build it? Well, as a learning tool. I thought it would be really cool if there was a system that isn't just
programmable, but also fully inspectable as well. And by building it on top of linux, it's possible to
use all the pre-existing tools to monitor and inspect the system while it's running.

---

## Getting started
The easiest way to start playing around is to clone the [playground](https://github.com/explore-os/play)
and opening it in vscode. The playground uses a devcontainer to set everything up properly and put you
directly into a working environment.

If you want to try something bigger or want to save your code in git, then its better to create a new repo
based on this. Thats why the project is configured as a template. Just use the green "Use this template" button
at the top to create your own playground

If you just want to play around a bit, you can just clone the repository and try it out:
```sh
git clone https://github.com/explore-os/play eos
code eos
```

When asked if you want to open the project inside a container, click yes so vscode can set up the docker
container for you.

Once vscode has started and built everything, you should be able to interact with the system through `eos`.

To start the provided test actor you can open a terminal in vscode and run the following command:
```sh
eos spawn --id test script /explore/examples/test-actor.rn
```

And to send it a message to see if its working you can use the following:
```sh
eos send /explore/actors/test '{"hello":"world"}'
```

You should see the message slowly traveling towards its destination and vanish, once the actor is done with it.
When that happens, you can use `cat` to output the contents of the `state` file in the actors directory,
which should contain the message tha was sent to it earlier.

To run your own code and start experimenting, you can write your own actor script in [rune](https://rune-rs.github.io/)
and spawn it through `eos`:
```sh
eos spawn /path/to/your/script
```

The rune script must contain the following function:
```rs
pub fn handle(state, msg) {
    // your code goes here

    // if you want to persist something, you can add it to an actors state
    state["detail"] = 10

    // the data you return gets persisted into the actors state file
    return state;
}
```

If the script wants to respond to a message, it can simply return the response along with the state:
```rs
pub fn handle(state, msg) {
    // your code goes here

    // if you want to persist something, you can add it to an actors state
    state["detail"] = 10

    let response = #{hello: "world"};

    // the data you return gets persisted into the actors state file
    return (state, response);
}
```

If the script contains a public function called `init`, it will call it after the actor started and
write the result of the init function the the actors state file.

e.g.:
```rs
pub fn init() {
    return #{initial_value: 10};
}
```

---

## Workspace

The devcontainer repo contains a directory called `workspace`, which gets mounted to `/explore/workspace`
and is intended for files that are part of the git repo containing the devcontainer, but need to be accessible
from inside the container.

This directory can be used to store all your scripts and files needed to replicate whatever scenario or use-case
you were testing or exploring. This allows you to commit those files and make it possible for others to clone
your repository and re-use the stuff you created.

---

## Persistence

EOS supports persistence via [redb](https://github.com/cberner/redb).

The `eos` lib exports a struct called `Db` which is a thin wrapper around redb. There are multiple subcommands
under `eos db` which can be used to access redb databases from the terminal. The script actor provides
the functions `store(name: string, key: string, value: any)`, `load(name: string, key: string)` and
`delete(name: string, key: string)` to the scripts, so actors can persist data across reboots.

The redb files are stored inside the directory `/explore/storage`. In the devcontainer this gets mounted to
an internal volume, allowing the files to survive restarts, while keeping them away from the workspace files.

---

## Teleplot

EOS supports plotting data via [teleplot](https://github.com/nesnes/teleplot).

The `eos` lib exports a function called `plot`, which can be used from inside an actor to send
data to a local teleplot server. The cli exposes that function as a subcommand (`eos plot`) to allow sending
data from the terminal as well.
The script actor provides the function `plot(data: string)` to allow scripts to send data for plotting as well.

The playground has the teleplot extension installed, which can be used to quickly
start a teleplot server and show the web interface directly inside vscode. This is done
by using the command palette (just type teleplot and it should show up), which will start a server and show
the teleplot UI. This is a manual action and must be done before any plot data is sent.

---

## How it works
EOS is an actor system that exposes its internal state throught the 9p protocol,
making it possible to interact with the system as if everything was a file.

To start the system you can use the command `eos start /mnt/eos`.
This will also automatically mount the system via 9p to `/mnt/eos`.

Once the system is running, its possible to interact with it through the `eos` cli tool
or by interacting with the files in the mounted filesystem.

Inside the mounted filesystem there is a directory called `actors`. After an actor is started,
a directory is created with the same id as the actor. This directory contains the actor's state
and can be used to interact with the actor.
