---
layout: post
title: "Hubot, Ubuntu and .deb packages"
date: 2012-11-25 15:58
comments: true
categories:
---

This week I finally got the time to check out [Hubot](http://hubot.github.com/).
Which really is a cool bot for your IRC/Campfire-Channel.

As I dislike solutions like "just extract this tarball and run ..." for
production deployment, I needed a way to deploy and configure this with
puppet. The solution should also be very easy, so we can create a
jenkins task to build the package.

I came out with a solution where you just need to run two commands to
create a new package from upstream:
```bash
rake prepare[2.3.2]
rake build
```

<!-- more -->

My solution was to use [fpm](https://github.com/jordansissel/fpm) to
create a debian package from a folder which contained all the stuff to
run Hubot.

As there is a huge repository of [Hubot-scripts](https://github.com/github/hubot-scripts)
and I also wanted some plugins out of it, I also needed a configuration
file to define which scripts I wanted to have packaged into.

To automate the process of downloading ```Hubot``` and the needed scripts,
afterwards installing the ```npm```-modules and packing it with fpm,
I decided to use ```rake```.

There is a configfile, ```config.yml```.

```ruby Configfile (config.yml)
package_url: 'https://github.com/downloads/github/hubot'
scripts_url: 'https://raw.github.com/github/hubot-scripts/master/src/scripts'
scripts:
 - jenkins
 - moarcatsme
 - redmine
```

The ```Rakefile``` contains all the rake-tasks which are used to build
the package.

```ruby Rakefile
require 'yaml'

desc "Prepare the package"
task :prepare, [:version] do |task,args|
  config = YAML.load_file('config.yml')

  sh "echo #{args.version} > VERSION"

  Dir.mkdir 'workdir'
  Dir.mkdir 'packages'

  Dir.chdir 'workdir'

  sh "curl -L #{config['package_url']}/hubot-#{args.version}.tar.gz | tar xzf -"

  config['scripts'].each do |script|
    sh "curl -L #{config['scripts_url']}/#{script}.coffee > scripts/#{script}.coffee"
  end

  sh 'npm install coffee-script'
  sh 'npm install hubot-irc --save'
  sh 'npm install'
end

desc 'Build the package'
task :build do
  Dir.chdir 'workdir'
  sh 'fpm -s dir -t deb -n hubot --prefix /opt -v $(cat ../VERSION) -d nodejs --after-install ../postinst --before-install ../preinst hubot'
  Dir.chdir '..'
  sh 'cp workdir/hubot_$(cat VERSION)_amd64.deb packages/'

  sh 'fpm -s dir -t deb -n hubot-init --prefix / -v $(cat VERSION) -d hubot,upstart --package packages/hubot-init-$(cat VERSION)_amd64.deb etc'
end
```

The ```prepare```-task takes one argument which should be the version (e.g ```2.3.2```).
It downloads the tarball of this version and extracts it in a subfolder.
Then it downloads all the specified scripts and puts them into the
correct folder of Hubot.
Afterwards it installs the ```npm```-modules to another subfolder of
hubot.

The most interesting part are the ```fpm```-lines:
```ruby fpm command to create the package
sh 'fpm -s dir -t deb -n hubot --prefix /opt -v $(cat ../VERSION) -d nodejs --after-install ../postinst --before-install ../preinst hubot'
```

This command creates a debian package (```-t deb```) from a directory
(```-s dir```) and throws all the stuff of the given directory
(last argument, ```hubot```) in ```/opt``` (```--prefix /opt```).
It uses the version we gave the ```prepare```-task
and adds a ```preinst``` and a ```postinst``` file.
With ```-d nodejs``` we set a dependency on ```nodejs```.
We need to change the current workingdir, otherwhise fpm would deploy
the stuff in ```/opt/workdir/hubot```.

The second fpm command packages the init script and the example config
file from the ```etc```-folder.
```ruby fpm command to package the init script and example config
sh 'fpm -s dir -t deb -n hubot-init --prefix / -v $(cat VERSION) -d hubot,upstart --package packages/hubot-init-$(cat VERSION)_amd64.deb etc'
```

The ```preinst``` and ```postinst``` scripts just add a ```hubot```
user and set the rights on ```/opt/hubot``` as fpm
[does not support right management for debian packages](https://github.com/jordansissel/fpm/issues/178),
yet.

I tried several times to get all these stuff working. For an easier cleanup
between the tries, I added another task:
```ruby cleanup task
require 'fileutils'

desc "Cleanup the workdir"
task :cleanup do
  FileUtils.rm_r 'workdir' if File.exists? 'workdir'
  FileUtils.rm_r 'packages' if File.exists? 'packages'
end
```

Now we just need to modify the ```config.yml```, push the changes and
call Hubot in our IRC-Channel to trigger the jenkins build:
```
hubot jenkins build hubot-build-deb, version=2.3.2
```
