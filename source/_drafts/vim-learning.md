---
title: Vim Learning Notes
tag: vim
---

## Get Started

### Installing Vim

Vim works in almost any OS environment, such Linux, Mac and Windows.

#### Mac

If you are using a Mac, vim is already installed and just skip this section.

#### Linux

For Ubuntu, just run this commmand in the terminal:

`sudo apt-get install vim`

For centos, just run:

`sudo yum install vim`

#### Windows

See the [Official Vim website](http://www.vim.org/download.php) to download. You can also use the [Cygwin](http://www.cygwin.com/ ) to simulate an Linux environment. But I strongly suggest you learn vim on Linux or Mac.

<!-- more -->

### Test Your Install

After you have finished the install, you can just type:

`vim --version`

If there terminal prints the version number of vim, you have successfully installed the vim. Now begin an brief introduction to vim.



### A brief introduction to vim

We can simply enter **Vim** to open it or attach with a file name:

`vim hello_vim.txt`

Now, you are using vim and we first need to know the two modes of Vim:

- **Command Mode:** enter some command such as saving file, quit vim, etc
- **Insert Mode:** edit text of the file

The vim start with command mode. You can simply type `i` to enter the insert mode and enter `ESC`

to exit insert mode to command mode.

Once we are in command mode, we need to first enter `:` before we can enter some command. We can enter `wq` (**w**rite and **q**uit) to save the changes and quit, enter `q!` to quit vim without saving changes. For more command, we will cover later.
