---
layout: post
title: "Vim with Ctags"
date: 2013-10-01 23:48
comments: true
categories: [Vim]
---

Today I setup my Vim to use [ctags](http://ctags.sourceforge.net/).

If you are deep in a project checking the backtrace of a bug and
are wondering what exactly a method call does, you can just hover it and
press `CTR + ]` to jump to the method definition.

If you are interested in such a setup, read on.

<!-- more -->

## Installing ctags

Install ctags with your package manager. On Mac OS X the best way is to use
[brew](http://brew.sh/):

 asd  brew install ctags

## Configuring the git hooks

As I didn't want to run ctags manually after every file change, I configured
some Git hooks (check [tpope's blog](http://tbaggery.com/2011/08/08/effortless-ctags-with-git.html) for more
background informations).

First, setup Git to use a templatedir when creating a repository or running `git init`.

```console
git config --global init.templatedir '~/.git_template'
mkdir -p ~/.git_template/hooks
```

Then create a script (`.git_template/hooks/ctags`) which calls `ctags` to reindex the current repository.
The content of this script looks like this:

```bash
#!/bin/sh
set -e
PATH="/usr/local/bin:$PATH"
trap "rm -f .git/tags.$$" EXIT
ctags --tag-relative -Rf.git/tags.$$ --exclude=.git --exclude=tmp --exclude=coverage --languages=-javascript,sql
mv .git/tags.$$ .git/tags
```

I already included the `exclude` directive for `tmp` and `coverage` which I
don't want to have indexed by ctags. Feel free to remove these or add any other.

Now, create all the Git hooks scripts (`.git_template/hooks/post-checkout`,
`.git_template/hooks/post-merge`, `.git_template/hooks/post-commit`) with the
following content:

```bash
#!/bin/sh
.git/hooks/ctags >/dev/null 2>&1 &
```


There is one special script (`.git_template/hooks/post-rewrite`) in which we
have to catch the rebase hook only:

```bash
#!/bin/sh
case "$1" in
  rebase) exec .git/hooks/post-merge ;;
esac
```

As these scripts save your `tags` file in `.git`, you don't have to worry about
adding it to `.gitignore` and its a project specific file. I wanted a project
specific index as I don't often search for methods which are saved in another
Git repository.
Another plus is, that the Vim plugin
[fugitive](https://github.com/tpope/vim-fugitive) (which I strongly suggest to
install) will automatically configure Vim for this file location.

## Setup the repos

If you want to install these hooks in an existing repository, just use `git
init` and it will copy them in.
For new repositories, they will be created automatically.

## Setup Vim

Vim has builtin support for `ctags` via its [tags
function](http://vim.wikia.com/wiki/Browsing_programs_with_tags), so none of the
following plugins are really needed.

### fugitive

Install plugin [fugitive](https://github.com/tpope/vim-fugitive) which helps you
a lot when writing stuff in a Git repository.

### Tagbar
Consider using [tagbar](https://github.com/majutsushi/tagbar) to have a sidebar
with the `ctags` of your current buffer.
In case you want to use `tagbar`, also think about opening it automatically. Put
the following configuration in your `vimrc`:

```vim
autocmd VimEnter * TagbarToggle
```

### Easytags
Maybe use [easytags](https://github.com/xolox/vim-easytags). It supports
reindexing the ctags upon saving the buffer.
If you want to use `easytags`, ensure that the configuration corresponds with
the setup above:

```vim
" ensure it checks the project specific tags file
let g:easytags_dynamic_files = 1
" configure easytags to run ctags after saving the buffer
let g:easytags_events = ['BufWritePost']
```

Happy debugging!
