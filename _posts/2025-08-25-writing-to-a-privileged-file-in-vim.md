---
layout: post
title: How to write to a Privileged File in Vim
categories: [Blog]
tags: [Vim, Linux]
description: How to run a command in Vim to save to a privileged file if you forgot to run Vim as sudo
---

So you are doing some kind of configuration on your Linux machine, such as editing the network interface or configuring PKI for SSH, and you find yourself in a file within the /etc/ directory. You make a bunch of changes, adding and editing a bunch of lines, and you go to run the command `:wq` to save the file and quit before you restart the systemd service. But wait, you forgot to run Vim as a privileged process! Now you think you will have to exit Vim and run it again with the `sudo` command in front, which will reset all of your changes that you spent a lot of time trying to configure. Maybe you remember what you changed from memory but more often than not you will have to go back to your research and redo all of your steps. But, what if I told you there is a convenient way to save to a privileged file in Vim? 

Throughout my time as a cybersecurity engineer, I've learned my way around Linux environments and how to navigate and make changes efficiently with the pre-installed CLI binaries (find, ls, grep, etc.), but I have constantly came across the issue where I forget to use `sudo` when I try to edit a file in the `etc` directory. Normally, I would do the tedious process where I exit out of the vim editor and redo the file changes but at some point I wanted to see if this process can be done more efficiently. Fortunately, one day I came across this [Stack Overflow post](https://stackoverflow.com/questions/2600783/how-does-the-vim-write-with-sudo-trick-work) that explains a way to save to a privileged file in Vim without losing your changes. 

The author of the solution in the post suggested that the command `:w !sudo tee %` should be used to save changes to a privileged file ran as an unprivileged user. Now there is a couple of differences between that command and the usual `:w` command that I'm going to go over.  

- First, the main similarity is that both commands are using `:w`. When Vim opens a file, it reads the contents of the file into a buffer stored in memory and all of the changes that happens within Vim is made within this buffer. When you call `:w`, it tells Vim to synchronize the changes made on the buffer with the actual file on disk to save the changes. However, if an unprivileged Vim process is trying to write to a privileged file, the system will prevent the process from syncing the changes made in the buffer with the file.
- The `!` command right before `sudo` is a special character that tells Vim to pipe the contents of the buffer to a shell command instead of writing the buffer to disk. 
- The `sudo` command is the standard command for elevating your current permissions from a regular user to a super user. It is used to run the shell command after the `!` character as a privileged user.
- `tee` is a binary found in many Linux distributions that reads data from standard input (in this case the Vim memory buffer) and writes the same content to both standard output and to a specified file, in this case being the file given by the special character `%`. 
- Finally, the special character `%` is a Vim variable that expands to the full path of the current file being modified. This ensures that `tee` is writing the file contents back to the same opened file, which is how this whole command is writing the file "in place". 

When you finally run the `:w !sudo tee %` command in Vim, it will ask you for your password and if your current user is in the sudo group the rest of the shell command will execute. When it executes, there will be a print of the edited file due to the way `tee` works, and there will be a warning that pops up because both the file on disk and the memory buffer has changed and it is trying to confirm if this is an intended action. This is a result of the external shell process we created in the command which edited the file and changed the file's modification timestamp while we have the other Vim process opened. This warning will look something like this:

```shell
W12: Warning: File "resolv.conf" has changed and the buffer was changed in Vim as well
See ":help W12" for more info.
[O]K, (L)oad File, Load File (a)nd Options:
```

To continue with the changes and save them, you want to choose the `[O]K` option, which tells Vim to ignore the warning and trust the current buffer. 
