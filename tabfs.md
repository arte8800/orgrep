---
title: TabFS
---
<!-- I'm setting this page in Verdana-on-gray so I feel more comfortable -->
<!-- jotting random notes and stuff down. -->
<style>
body { font-family: Verdana; background: #eee; }
h1 { font-family: Helvetica; }
</style>

[TabFS](https://github.com/osnr/TabFS) is a browser extension that
mounts your browser tabs as a filesystem on your computer.

Out of the box, it supports Chrome and (to a lesser extent[^firefox])
Firefox, on macOS and Linux.[^otherbrowsers]

[^firefox]: because of the absence of the [chrome.debugger API for
    extensions](https://developer.chrome.com/docs/extensions/reference/debugger/).
    With a bit more plumbing, you could maybe find a way to connect it
    to the remote debugging protocol in Firefox and other browsers and
    get that second level of functionality that is currently
    Chrome-only.

[^otherbrowsers]: It could probably be made to work on other browsers
    like Safari and Opera that support the WebExtensions API, and on
    Windows using Dokan or WinFUSE/WSL stuff, but I haven't looked
    into that. 

Each of your open tabs is mapped to a folder.

<div class="figure">
<img src="doc/finder.png">
</div>

The files inside a tab's folder directly reflect (and can control) the
state of that tab in your browser. (TODO: update as I add more)

<div class="figure">
<img src="doc/finder-contents.png">
</div>

This gives you a _ton_ of power, because now you can apply [all the
existing tools](https://twitter.com/rsnous/status/1018570020324962305)
on your computer that already know how to deal with files -- terminal
commands, scripting languages, etc -- and use them to control and
communicate with your browser.

Now you don't need to [code up a browser extension from
scratch](https://twitter.com/rsnous/status/1261392522485526528) every
time you want to do anything. You can write a script that talks to
your browser in, like, a melange of Python and bash, and you can save
it as [a single ordinary
file](https://twitter.com/rsnous/status/1308588645872435202) that you
can run whenever, and it's no different from scripting any other part
of your computer.

## Examples of stuff you can do!

(assuming your current directory is the `fs` subdirectory of the git
repo and you have the extension running)

(TODO: more of these)

### List the titles of all the tabs you have open 

```
$ cat mnt/tabs/by-id/*/title.txt
GitHub
Extensions
TabFS/install.sh at master · osnr/TabFS
Alternative Extension Distribution Options - Google Chrome
Web Store Hosting and Updating - Google Chrome
Home / Twitter
...
```

### Close all Stack Overflow tabs

```
$ rm mnt/tabs/by-title/*Stack_Overflow*
```

or (older / more explicit)

```
$ echo remove | tee -a mnt/tabs/by-title/*Stack_Overflow*/control
```

#### btw

(this task, removing all tabs whose titles contain some string, is a
little contrived, but it's not that unrealistic, right?)

(now... how would you do this _without_ TabFS?  I honestly have no
idea, off the top of my head. like, how do you even get the titles of
tabs? how do you tell the browser to close them?)

(I looked up the APIs, and, OK, if you're already in a browser
extension, in a 'background script' inside the extension, _and_ your
extension has the `tabs` permission -- this already requires you to
make 2 separate files and hop between your browser and your text
editor to set it all up! -- you can do
[this](https://developer.chrome.com/docs/extensions/reference/tabs/#method-query):
`chrome.tabs.query({}, tabs => chrome.tabs.remove(tabs.filter(tab =>
tab.title.includes('Stack Overflow')).map(tab => tab.id)))`)

(not _terrible_, but look at all that upfront overhead to get it set
up. and it's not all that discoverable. and what if you want to reuse
this later, or plug it into some larger pipeline of tools on your
computer, or give it a visual interface? the jump in complexity once
you need to communicate with anything -- possibly setting up a
WebSocket, setting up handlers and a state machine -- is pretty
horrifying)

(but to be honest, I wouldn't even have conceived of this as a thing I
could do in the first place)

### Save text of all tabs to a file

```
$ cat mnt/tabs/by-id/*/text.txt > text-of-all-tabs.txt
```

### Reload an extension when you edit its source code

Suppose you're working on a Chrome extension (apart from this
one). It's a pain to reload the extension (and possibly affected Web
pages) every time you change its code. There's a [Stack Overflow
post](https://stackoverflow.com/questions/2963260/how-do-i-auto-reload-a-chrome-extension-im-developing)
with ways to automate this, but they're all sort of hacky. You need
yet another extension, or you need to tack weird permissions onto your
work-in-progress extension, and you don't just get a command you can
trigger from your editor or shell.

TabFS lets you subsume all this into [an ordinary shell
script](https://github.com/osnr/playgroundize-devtools-protocol/blob/main/go.sh),
with no need for any further browser-side code.

The script linked above turns the extension (this one's title is
"Playgroundize DevTools Protocol") off, then turns it back on, then
reloads any Chrome DevTools pages that are open:

```
#!/bin/bash -eux
echo false > mnt/extensions/Playg*/enabled
echo true > mnt/extensions/Playg*/enabled
echo reload | tee mnt/tabs/by-title/Chrome_Dev*/control
```

I mapped this script to Ctrl-. in my Emacs, so now I can just hit that
every time I want to reload my extension code!

### TODO: Cull tabs like any other files 

I do this in Emacs dired.

### TODO: Live edit a running Web page


(TODO: it would be cool to have a persistent storage story here)

### TODO: Something with live view of variables

two floating windows. watch
mouse position?

### TODO: Import data (JSON or XLS). Persistent storage?

hehehehehehehehehe

import a plotting library too?

## Setup

**disclaimer**: this extension is an experiment. I think it's cool and
useful and provocative, and I usually leave it on, but I make no
promises about functionality or, especially, security. applications
may freeze, your browser may freeze, there may be ways for Web pages
to use the extension to escape and hurt your computer ... In some
sense, the [whole
point](https://twitter.com/rsnous/status/1338932056743546880) of this
extension is to create a gigantic new surface area of communication
between stuff inside your browser and software on the rest of your
computer.

First, install the browser extension.

Then, install the C filesystem.

### 1. Install the browser extension

(I think for Opera or whatever other Chromium-based browser, you could
get it to work, but you'd need to change the native messaging path in
install.sh. Not sure about Safari. maybe Edge too? if you also got
everything to compile for Windows)

#### in Chrome

Go to the [Chrome extensions page](chrome://extensions). Enable
Developer mode (top-right corner).

Load-unpacked the `extension/` folder in this repo.

**Make a note of the extension ID Chrome assigns.** Mine is
`jimpolemfaeckpjijgapgkmolankohgj`. We'll use this later.

#### in Firefox

You'll need to install as a "temporary extension", so it'll only last
in your current FF session. (TODO: is this fixable? signature stuff?)

Go to [about:debugging#/runtime/this-firefox](about:debugging#/runtime/this-firefox).

Load Temporary Add-on...

Choose manifest.json in the extension subfolder of this repo.

### 2. Install the C filesystem

First, make sure you `git submodule update --init` to get the
`fs/frozen` dependency.

And make sure you have FUSE and FUSE headers. On Linux, for example,
`sudo apt install libfuse-dev` or equivalent. On macOS, get [FUSE for
macOS](https://osxfuse.github.io/).

Then compile the C filesystem:

```
$ cd fs
$ mkdir mnt
$ make
```

Now install the native messaging host into your browser, so the
extension can launch and talk to the filesystem:

#### Chrome and Chromium

Substitute the extension ID you copied earlier for
`jimpolemfaeckpjijgapgkmolankohgj` in the command below.

```
$ ./install.sh chrome jimpolemfaeckpjijgapgkmolankohgj
```

or

```
$ ./install.sh chromium jimpolemfaeckpjijgapgkmolankohgj
```

### 3. Ready!

Go back to `chrome://extensions` or
`about:debugging#/runtime/this-firefox` and reload the extension.

Now your browser tabs should be mounted in `fs/mnt`!

Open the background page inspector to see the filesystem operations
stream in. (in Chrome, click "background page" next to "Inspect views"
in the extension's entry in the Chrome extensions page; in Firefox,
click "Inspect")

<div class="figure">
<img style="max-width: 90%; max-height: 1000px" src="doc/inspector.png">
</div>

This console is also incredibly helpful for debugging anything that
goes wrong, which probably will happen.

(My OS and applications are pretty chatty! They do a lot of
operations, even when I don't feel like I'm actually doing anything.)

## Design

- `fs/`: Native FUSE filesystem, written in C
  - [`tabfs.c`](https://github.com/osnr/TabFS/tree/master/fs/tabfs.c):
    Talks to FUSE, implements fs operations, talks to extension. I
    rarely have to change this file; it essentially is just a stub
    that forwards everything to the browser extension.
- `extension/`: Browser extension, written in JS
  - [`background.js`](https://github.com/osnr/TabFS/tree/master/extension/background.js):
    **The most interesting file**. Defines all the synthetic files and
    what browser operations they invoke behind the scenes.[^frustrates]

[^frustrates]: it frustrates me that I can't have something like a
    table of contents for this source file. because it does have a
    structure to it! so I feel like the UI for looking at the file
    should highlight and exploit that structure.
    
    I want to link you to a particular route and talk about it here
    and also have some kind of
    transclusion (without the horrifying mess of making a lot of tiny
    separate files). I want to use typesetting and whitespace to set
    each route in that file apart, and set them as a whole apart from the utility functions &
    default implementations & networking.

My understanding is that when you, for example, `cat
mnt/tabs/by-id/6377/title.txt` in the tab filesystem:

1. `cat` on your computer does a system call `open()` down into macOS
   or Linux,

2. macOS/Linux sees that this path is part of a FUSE filesystem, so it
   forwards the `open()` to the FUSE kernel module,

3. FUSE forwards it to the `tabfs_open` implementation in our
   userspace filesystem in `fs/tabfs.c`,

4. then `tabfs_open` rephrases the request as a JSON string and
   forwards it to our browser extension over stdout (['native
   messaging'](https://developer.chrome.com/docs/apps/nativeMessaging/)),

6. our browser extension in `extension/background.js` gets the
   incoming message; it triggers the route for
   `/tabs/by-id/*/title.txt`, which calls the browser extension API
   `browser.tabs.get` to get the data about tab ID `6377`, including
   its title,

7. so when `cat` does `read()` later, the title can get sent back in
   a JSON native message to `tabfs.c` and finally back to FUSE and the
   kernel and `cat`.

(very little actual work happened here, tbh. it's all just
marshalling)

TODO: make diagrams?

## License

GPLv3

## things that would be good to do

- multithreading. the key constraint is that I give `-s` to
  `fuse_main` in `tabfs.c`, which makes everything
  single-threaded. but I'm not clear on how much it would improve
  performance? maybe a lot, but not sure. maybe workload-dependent?

    the extension itself (and the standard I/O comm between the fs and
  the extension) would still be single-threaded, but you could
  interleave requests since most of that stuff is async. like the
  screenshot request that takes like half a second, you could do other
  stuff while waiting for the browser to get back to you on that (?)

    another issue is that _applications_ tend to hang if any
  individual request hangs anyway; they're not expecting the
  filesystem to be so slow (and to be fair to them, they really have
  [no way](https://twitter.com/whitequark/status/1133905587819941888)
  to). some of these are problems that may be inevitable for any FUSE
  filesystem, even ones you'd assume are reasonably battle-tested and
  well-engineered like sshfs?

- look into support for Firefox / Windows / Safari / etc. best FUSE
  equiv for Windows? can you bridge to the remote debugging APIs that
  all of them have to get the augmented functionality? or just
  implement it all with JS monkey patching?

- fix leaks

- elim unnecessary round-trips / browser API calls

## hmm

- there's a famous (?) paper, [Processes as Files
(1984)](https://lucasvr.gobolinux.org/etc/Killian84-Procfs-USENIX.pdf),
which lays out the case for the `/proc` filesystem. it's very cool!
very elegant in how it reapplies the existing interface of files to
the new domain of Unix processes. but how much do I care about Unix
processes now? most
[programs](https://twitter.com/rsnous/status/1176587656915849218) that
I care about running on my computer these days are Web pages, [not
Unix
processes](https://twitter.com/rsnous/status/1076229968017772544). so
I want to take the approach of `/proc` -- 'expose the stuff you care
about as a filesystem' -- and apply it to something
[modern](https://twitter.com/rsnous/status/1251342095698112512): the
inside of the browser. can we have 'browser tabs as files'?

- there are two 'operating systems' on my computer, the browser and
Unix, and Unix is by far the more accessible and programmable and
cohesive as a computing environment (shells, processes), even though
it's arguably the less important to my daily life. how can the browser
take on these properties?

- it's [way too
hard](https://twitter.com/rsnous/status/1342236988938719232) to make a
browser extension. even 'make an extension' is a bad framing; it
suggests making an extension is a whole Thing, a whole Project. like,
why can't I just take a minute to ask my browser a question or tell it
to automate something? lightness

- open input space -- filesystem. (it reminds me of Screenotate.)

- now you have this whole 'language', this whole toolset, to control
and automate your browser. there's this built-up existing capital
where lots of people and lots of application software and lots of
programming languages ... already know the operations to work with
files

- this project is cool bc i immediately get [a dataset i care
  about](https://twitter.com/rsnous/status/1084166291793965056). I
  found myself using it 'authentically' pretty quickly -- to clear out
  my tabs, to help me develop other things in the browser so I'd have
  actions I could trigger from my editor, ...

- SQLite. OSQuery

- fake filesystems talk

- [rmdir a non-empty
  directory](https://twitter.com/rsnous/status/1107427906832089088) I
  feel like a new OS, something like Plan 9, should
  [generalize](https://twitter.com/rsnous/status/1070830656005988352)
  its file I/O APIs just enough to avoid problems like this. like
  design them with the disk in mind but also a few concrete cases
  of synthetic filesystems, very slow remote filesystems, etc

do you like setting up sockets? I don't

https://luciopaiva.com/witchcraft/