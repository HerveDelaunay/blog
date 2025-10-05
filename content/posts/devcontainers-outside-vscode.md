+++
title = 'Building portable development environments with Devpod'
date = 2025-10-05T16:53:15+02:00
draft = false
+++

I've been looking for a way to create reproducible development environments without relying on VSCode.

Devcontainers are awesome, they give me:

- a fully reproducible development environment, and thus tremendous consistency
- a configuration that can be shared with the whole team, since it’s declared in a simple JSON file
- tool isolation - each project can have its own stack, without conflicts
- and most importantly, my machine stays clean of all this tooling

The only problem I have with devcontainers is that they rely on VSCode to run.
As a CLI-first engineer, and as a Vim user, I had to find another way.

**Enter: Devpod.**

## What is Devpod

Devpod allows me to open devcontainers in different ways, including directly from the CLI.

It supports several "providers", which are basically the backends where your container will run.

You can use:

- **Docker**, to run it locally
- **Kubernetes**, if you want to run your environment in a remote cluster
- **Cloud providers**, if your laptop isn’t powerful enough for the job

Once your provider is set up, you can also choose _how_ to open the workspace:

- with VSCode
- with JetBrains IDEs
- or **with no IDE at all** - which is what I prefer.

## Spinning up a Devcontainer

It works like this:

You create a `.devcontainer/` directory at the root of your project, and inside it a single file: `.devcontainer.json`.

That file is a **declarative description** of your development environment - it defines the image, the tools, and the ports you want to expose.

Once it’s in place, you can simply run:

```bash  
devpod up . --ide none
```

…and Devpod will build and start your container.

## A fully personalized environment

One of the neat features of Devpod is that it can automatically clone your dotfiles repository when creating a new container:

```bash
# Devpod will clone the repo to ~
devpod up . --ide none --dotfile git@github.com:HerveDelaunay/dotfiles.git
```

If the repository contains a script named `setup`, Devpod will execute it right after cloning.
This is a powerful mechanism: it lets you customize your environment as soon as the container starts.

Typically, people use this setup script to create symlinks for their dotfiles or configure their shell.
In my case, I use it to bootstrap my environment in a slightly different way (more on that in future articles).

That’s the beauty of Devpod: it gives you a reproducible, containerized workspace that already feels like your own system.

## Exploring Devpod in practice

Once your container is up, you can list all your running or existing Devpods with:

```bash  
devpod ls
```

It gives you an overview of your environments - one per project.
Each entry shows where the workspace is located, the provider used (Docker, Kubernetes, etc.), and whether an IDE is attached or not.

For example:

```bash  
❯ devpod ls

NAME | SOURCE | MACHINE | PROVIDER | IDE | LAST USED | AGE | PRO

fastapi | local:/Users/herve/repos/github.com/HerveDelaunay/fastapi | | docker | none | 3d2h | 5d4h | false

study | local:/Users/herve/repos/github.com/HerveDelaunay/study | | docker | none | 1h5m | 2d3h | false
```

Each line here represents a ready-to-use development environment.

If I want to get into one of them, I can simply SSH into it:

```bash  
devpod ssh fastapi
```

This drops me directly inside the `/workspaces/fastapi` directory - which corresponds to the project’s root folder on my host machine.

## Setting default behavior

Devpod can integrate with a long list of IDEs, but since I prefer working directly in the terminal,
I set my default IDE to `none` once and for all:

```bash  
devpod ide use none
```

You can check your current setting with:

```bash
❯ devpod ide list

         NAME       | DEFAULT
  ------------------+----------
    clion           | false
    codium          | false
    cursor          | false
    dataspell       | false
    fleet           | false
    goland          | false
    intellij        | false
    jupyternotebook | false
    none            | true
    openvscode      | false
    phpstorm        | false
    positron        | false
    pycharm         | false
    rider           | false
    rstudio         | false
    rubymine        | false
    rustrover       | false
    vscode          | false
    vscode-insiders | false
    webstorm        | false
    zed             | false
```

From now on, Devpod won’t try to launch any graphical editor when you spin up a new workspace - it’ll just open the container.

This is perfect for CLI-first workflows or Vim-based setups.

It’s also possible to set your dotfiles repository as the default one, so you don’t have to specify it every time you run devpod up:
```bash
# Set default dotfiles repo
devpod context set-options -o DOTFILES_URL=git@github.com/HerveDelaunay/dotfiles.git
```

Now every new workspace will automatically use your personal configuration without any extra flags.

## Opening Dev environments in the browser

If you sometimes need a full IDE, Devpod can also open a browser-based instance of VSCode (via OpenVSCode):

```bash  
devpod up . --ide openvscode --dotfiles git@github.com/HerveDelaunay/dotfiles.git
```

This launches a complete editor environment accessible directly in your browser.
It’s especially handy when you’re on a machine that doesn’t have VSCode installed - you still get a full coding experience with your custom config and theme.

## Wrapping up

Devpod brings the devcontainer concept outside of VSCode, making it accessible to CLI users, remote clusters, and cloud workspaces.

With a simple command, you can reproduce your entire development environment - with your dotfiles, tools, and configs - anywhere.

## Resources

- [Devpod Documentation](https://devpod.sh/docs)
- [Devcontainers Specification](https://containers.dev/)
- [My dotfiles on GitHub](https://github.com/HerveDelaunay/dotfiles)
