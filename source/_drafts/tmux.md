---
title: An introduction to tmux (a terminal multiplexer)
tags: linux, tmux
---

Lots of people love working with the command line locally or on the server. A serious problem is that one terminal is always not enough. And the procedure started by the terminal is depended on the terminal and it will be killed when the terminal exit. This problem can be more serious especially when we work on a server when the terminal can be always disconnected because of the poor network environment. **Tmux** is such a terminal multiplexer which allows you to create multiple windows and panes within a single terminal window and it keeps all windows in a session which can be kept alive until you kill the tmux server.

# Installation

## Mac

For Mac user, you can just type in the terminal `brew install tmux`

## Ubuntu

`sudo apt-get install tmux`

## Centos

`sudo yum install tmux`

## Others

For the users working on other plateform or you want to build from the source, follow [this](https://github.com/tmux/tmux) to build from the source and install tmux.

