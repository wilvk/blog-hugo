+++
title = "Vim Plugin Development Workflows"
date = 2020-06-03T22:36:37+10:00
draft = false
tags = []
categories = []
+++

I use a Vim Plugin Manager called [Vundle](https://github.com/VundleVim/Vundle.vim) to install and update all my Vim plugins.

When developing a new plugin with Vim, it is useful to be able to develop locally, test the changes in Vim and be able to push the changes to a Git repository.

## Pushing to a remote repo and refreshing the local plugin

A naïve approach to this is to create a Git repository for your plugin, add it as a Vundle plugin in your `.vimrc`file and do development in a separate copy of the repository.

e.g.

```vim
    Plugin 'wilvk/NERDTreeWindowResizer'
```

This approach has the additional overhead of having to:

Commit and push changes from the local source folder (e.g. `~/source/wilvk/<plugin_name>`

Re-install or update the plugin in the `~/.vim/bundle/<plugin_name>` path via the Vim command `:PluginInstall` or `:PluginUpdate` commands.

This can become quite tedious if you are doing many changes and want to keep synchronised between the local source path and the Vim plugin.

## Referencing a local repo and creating a git remote for pushing to

An alternative approach I have found is, after creating your plugin's repository, to:

- Clone the plugin repository to your source path as you normally would:

```bash
$ cd ~/source/wilvk
$ git clone https://github.com/wilvk/NERDTreeWindowResizer
```

- Then, make a symbolic link to your plugin source code in the `~/.vim/bundle` path.

```bash
$ cd ~/.vim/bundle
$ ln -s ~/source/wilvk/NERDTreeWindowResizer ~/.vim/bundle/NERDTreeWindowResizer
```

- Back in the plugin source path, we need to make changes to our local repository remotes:

Firstly, we change to our source code path and see what our current remotes are with `git remote -v`:

```bash
$ cd ~/source/wilvk/NERDTreeWindowResizer
$ git remote -v
origin  https://github.com/wilvk/NERDTreeWindowResizer (fetch)
origin  https://github.com/wilvk/NERDTreeWindowResizer (push)
```

We then set the `origin` to a local path in the `~/.vim/bundle` path.

```bash
$ git remote set-url origin file:///Users/willvk/.vim/bundle/NERDTreeWindowResize
```

And create an `upstream` remote as our remote Git repository.

```bash
$ git remote add upstream https://github.com/wilvk/NERDTreeWindowResizer
```

We can verify the remotes are set correctly by running `git remote -v` again.

```bash
$ git remote -v
origin  file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer (fetch)
origin  file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer (push)
upstream        https://github.com/wilvk/NERDTreeWindowResizer (fetch)
upstream        https://github.com/wilvk/NERDTreeWindowResizer (push)
```

In our `.vimrc` file, we can add the plugin with a `Plugin` call, and reference the local repository:

```vim
call vundle#begin()
  ...
  Plugin 'file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer'
  ...
call vundle#end()
```

Then finally, in Vim, run `:PluginInstall` to make sure it is installed correctly.

Now, when you make changes to your local plugin's source, the changes will reflect instantly in Vim.
To push changes to the remote Git repository, you should now push with `git push upstream` as the `origin` remote is being used to keep Vundle in sync (which will always be in sync now as it is a local file repository).

## Errors

If you get an error like the following when starting Vim:

```bash
Error detected while processing function vundle#config#bundle[2]..<SNR>7_check_bundle_name:
line    2:
Vundle error: Name collision for Plugin file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer.
Plugin wilvk/NERDTreeWindowResizer previously used the name "NERDTreeWindowResizer".
Skipping Plugin file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer.
Press ENTER or type command to continue
```

Make sure you have removed any previous reference to your plugin in your `.vimrc` file.

e.g.

```vim
call vundle#begin()
  ...
  Plugin 'wilvk/NERDTreeWindowResizer'
  Plugin 'file:///Users/willvk/.vim/bundle/NERDTreeWindowResizer'
  ...
call vundle#end()
```

In the above `.vimrc`, the first call to `Plugin` should be removed.

And that's it! Happy Vimming!
