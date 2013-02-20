---
layout: post
title: "Vim inline git diff"
date: 2013-02-20 22:23
comments: true
categories: ['Vim', 'Ruby', 'Git']
---

A few weeks ago, I heard about the Sublime Text Plugin [GitGutter](https://github.com/jisaacks/GitGutter).
When I googled for an equivalent plugin for Vim, I did not find anything usable.
There is [Svndiff](http://www.vim.org/scripts/script.php?script_id=1881), but I
did not like the name and that you had to enable it by hand. So I put the topic
aside and went on with other stuff.

Last week, I started over with my Vim configuration. I was using the
[dotfiles of skwp](https://github.com/skwp/dotfiles) which included also a
pretty nice Vim configuration. But after working with it for some months, I
started to feel uncomfortable with it as it had just too much features.

As part of building up my Vim configuration, I also checked again the topic
of inline git annotations.
I googled once more and found out, that these annotations are called ```signs```
in Vim. I also found the plugin of [sickill](https://github.com/sickill/vim-git-inline-diff).
This plugin uses ruby (I did not even know, that is possible..!) to do the core work of
parsing the diff and set the correct marks. Unfortunately it did not work for me...
But hey, this is ruby. Let's fix this!

After some hours of learning how to read ```diff -u``` output and tuning the algorithm to
mark the correct lines, I had a working version of ```vim-git-inline-diff```.

I made a [fork](https://github.com/luxflux/vim-git-inline-diff) of the existing plugin
and added a ```Readme```. There is also a possibility to configure the used symbols, now:

![Screenshot](https://github.com/luxflux/vim-git-inline-diff/raw/master/_assets/example.png)

Happy coding!

