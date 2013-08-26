---
layout: post
title: The Effect of Non-Breaking Spaces on Typography
---

I recently read [*Butterick's Practical Typography*](http://practicaltypography.com/), which introduced  me to the concept of [non-breaking spaces](http://practicaltypography.com/nonbreaking-spaces.html).  

>A *non­break­ing space* is the same width as a word space, but it pre­vents the text from flowing to a new line or page. It’s like in­vis­i­ble glue be­tween the words on ei­ther side.

Screenshot of Butterick's section/paragraph mark example

Butterick's paragraph and section mark example gives a sure use case for when to use a non-breaking space, but after seeing a blog post break OS<br>X onto separate lines, I decided to explore how more extensive use of non-breaking spaces affected typography.

One of my first tests was making the first and last spaces of a sentence non-breaking. This didn't seem to have much effect, and sometimes worsened the situation by e.g. causing a break between an adverb and adjective, e.g. very<br> tolerant.

// Screenshot of that

But using breaking spaces on a list of whitelisted words works well (does it?)

// Screenshot of that

// Effects on different browsers/screen sizes

I'm not sure how much non-breaking spaces will improve my typography, but given the permutations of browsers and screen sizes, manually inserting non-breaking spaces is an unreliable fix. I wrote a small script to replace a list of whitelisted words with

### Manual vs Scripted NBSP

The value of running a script to apply NBSPs is dependent on your site layout. If your line length is constant If you have a fancy dynamic layout that changes as the o

### OS X Service

One way to use the script is as an OS X service. This will allow you to right click in any Cocoa application (unfortunately, not Sublime Text) and replace replace the spaces with non-breaking spaces in your whitelist.

1. Open Automator
2. *Choose Service*
3. Check the *Output replaces selected text* option
4. Drag the *Run Shell Script* action from the list on the left into the editor
5. Paste in the following to call the script, replacing the path to Python3 and to the script as necessary:

  ```
  export PYTHONIOENCODING=UTF-8
/usr/local/bin/python3 /Users/Max/Documents/Python/Scripts/nbspPreprocess.py "$@"
  ```

6. Set "Pass Input" to *as arguments* (note: maybe if I use stdin I can just read from that in python? not sure)
7. Save

While redesigning this site, I noticed that even on the same laptop that Chrome, Firefox, and Safari all had different line lengths. 

