+++
title = 'How to Crack Zip Files on macOS'
date = 2023-06-02
+++

Because for some reason it's way too complicated.

![GIF of man swinging keyboard at his computer monitor.](pebkac.gif)

TL;DR

```
brew install fcrackzip

fcrackzip -b -l 1-100 file.zip

# -b    brute force
# -l    password length range
#
# fcrackzip --help  for more options
```

**Yep. I know, you tried installing John the Ripper but cannot, for the life of you, ever actually obtain the program zip2john.**

Well there’s the solution, and it’s pretty fast. I saved you having to read my pity story but here’s what lead to me finding that:

## The Story

It started months ago. I found this file on my Google Drive called “code-lol.zip”. And I’m thinking, “what the hell, why would I say ‘lol’ after code?” So I tried to open it on my computer and there’s the password box, I try every possible password I could’ve ever used. I probably made a list of the ones I tried, attempted a dozen passwords. But to no avail.

I looked up “how to crack a zip file” and got an awesome blog post explaining it perfectly. They say to use the John the Ripper community enhanced version program “zip2john” to obtain the hash of the zip file and then crack that hash using hashcat. Well, if I was on Linux like all these other nerds, it would’ve worked. But after a `brew install john` I got “program not found.” zip2john wasn’t there.

Then I learned about brew install john-jumbo and how it should come with extended features. Unless I was missing some important detail, there was still no zip2john. What the actual \*\*\*\*!

I came up with the ultimate, genius realization. Maybe I should watch the first YouTube video that comes up, instead. And I did just that, and the creator of the video is the same creator of fcrackzip, a program that can solve all your horrors. And in record speed, as well.

It nearly instantly said:

```
possible password: buddy
```

I let it run for a little while longer before I killed the process. I wondered why I would ever use such a simple password.

I go to that zip file with my uncertain password in hand, and I open the gate to find...

Two albums of mp3 files that I didn’t want anyone to know I pirated. I sent the zip file to my best friend over Discord. And I called it “code-lol.zip” because I didn’t want people to know it was music.

I hate my past self.

And I think I hate my current self for taking the time to find that.

So overall, please go high five the creator of fcrackzip, because that saved my butt and it probably saved your butt, if SEO is anything useful.
