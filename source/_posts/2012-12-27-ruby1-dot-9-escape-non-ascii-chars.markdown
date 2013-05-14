---
layout: post
title: "Ruby1.9: Escape non-ASCII chars"
date: 2012-12-27 20:47
comments: true
categories: [Ruby, Ruby1.9, ASCII, Unicode]
---

What I learned recently about Unicode and Ruby. There is a [TL;DR](/blog/2012/12/27/ruby1-dot-9-escape-non-ascii-chars/#tldr).

<!-- more -->

At nine.ch, we have a webinterface to administrate our mailboxes. It also can be used to
configure (Sieve-)filters for incoming mails. These filters are persisted in a database and
uploaded to the storage via the Managesieve protocol.

As we wanted to migrate to a new version of storage software, the uploading of these filters started to fail for some mailboxes.
The error was something like ```SieveError PUTSCRIPT: Too many arguments```.

Yeah, what does this mean...? [The RFC says](http://tools.ietf.org/html/draft-martin-managesieve-12#section-2.6)
that the syntax for ```PUTSCRIPT``` is like the following:

```
Putscript "mysievescript" {110+}
require ["fileinto"];

if envelope :contains "to" "tmartin+sent" {
  fileinto "INBOX.sent";
}
```

A failing upload looked like this:
```
Putscript "mysievescript" {823+}
require ["fileinto","vacation","body","date","relational"];

if allof(not header :contains ["X-Spam-Flag"] "YES") {
  vacation :days 3 :from "'Name FamilyName' <name@domain.tld>" :subject "Abwesenheitsmeldung" "sehr geehrte damen und herren
unser büro ist wegen ferienabwesenheit geschlossen. in dringenden fällen erreichen sie uns unter ....
}

# Rule Nr. 28804, sort_position 1
if header :contains ["X-Spam-Flag"] "YES" {
  fileinto "INBOX.Spam";
  stop;
}
.
.
.
```

So this should work, no? Nope, doesn't. Hmm, there are umlauts in this vacation message... Trial and error shows, that removing them helps and
the script is accepted by the server.

Cool, looks like the server does something strange with the string containing the script and thinks you give him more arguments than allowed.

Now, how shall I fix this? During my research I stumbled over the extension list for [Pigeonhole Sieve](http://wiki2.dovecot.org/Pigeonhole/Sieve).
There is an extension called ```encoded-character```, maybe this helps? Let's try.

We "just" have to escape these special chars according [the RFC](http://tools.ietf.org/html/rfc5228#section-2.4.2.4).

A first try with the following code gave me the broken characters (Ã¼, Ã¤, ...) in the vacation answer, known from UTF-8/ISO problems.

```ruby
def text
  clean_text = ''
  @text.join("\n").each_byte do |byte|
    unless byte > 127
      clean_text << byte
    else
      clean_text << "${UNICODE:#{byte.to_s(16)}}"
    end
  end
  clean_text
end
```

Relevant part from the sieve script:
```text
unser b${UNICODE:c3}${UNICODE:bc}ro ist wegen ferienabwesenheit geschlossen.
in dringenden f${UNICODE:bc}${UNICODE:e4}llen erreichen sie uns unter ....
```

I shortly googled an [utf-8 chartable](http://www.utf8-chartable.de/) and checked the content of the script.
Looks like my simple ```ü``` and ```ä``` are two bytes in Unicode? And when we just translate one byte at a time, this gives us
these (hated) character-combinations as each byte is interpreted as a single Unicode character?
Heard about this, but never really thought about it before.

Okay, lets consult the Ruby String documentation and check whether there is a better method to get these characters:
[String#each_codepoint](http://www.ruby-doc.org/core-1.9.3/String.html#method-i-each_codepoint).

Now it works!

Final version of the method to use ```each_codepoint```:

```ruby
def text
  clean_text = ''
  @text.join("\n").each_codepoint do |codepoint|
    unless codepoint > 127
      clean_text << codepoint
    else
      clean_text << "${UNICODE:#{codepoint.to_s(16)}}"
    end
  end
  clean_text
end
```

Relevant part from sieve script:
```text
unser b${UNICODE:fc}ro ist wegen ferienabwesenheit geschlossen.
in dringenden f${UNICODE:e4}llen erreichen sie uns unter ....
```


# <a id='tldr'></a> TL;DR

Use ```String#each_codepoint``` if you read a string and want to use the hex representation of its characters. Otherwhise you create
an encoding problem without changing the encoding.
