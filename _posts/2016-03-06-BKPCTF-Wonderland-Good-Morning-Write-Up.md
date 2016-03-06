---
layout: post
title: BKP CTF - Good Morning (Wonderland)
date: 2016-03-06 00:00:00.000000000 -08:00
author: Peter Adkins
image: 
---
Over the last two days I've been participating in the Boston Key Party (BKP) CTF with a group ephemerally known as 'Fear Of A Whitehat Planet'. In the end, we didn't do *too* badly - with all of the web challenges, a couple of crypto, and only one of the pwn challenges complete - but better luck next time, eh? 

The first web application that we worked on was 'Good Morning' (Wonderland on the CTF console). Although not a particularly complex challenge in the end, I thought I'd write-up our solution all the same.

### Good Morning!

![Nobody expects the Spanish inquisition](/assets/article_images/2016/monty.png)

When first opened, Good Morning presented the user with a number of seemingly Python inspired questions:

* What is your name?
* What is your quest?
* What is your favourite color?

Once all questions were answered, a thank you would be displayed with a survey number included in the response. When viewed through OWASP Zap, it was quickly apparent that all requests past the initial page load were over a WebSocket back to the origin.

### The Setup

As the source for the application was provided, we decided to have a quick dig to see what's happening on the server side.

The first thing which one of our team members (`jdoe`) noticed was that the MySQL character-set was hard-set to `Shift-JIS`, and input sanitization was being performed using a special character array and `s.replace`. As a result, it was almost guaranteed that this challenge would be able to be solved via SQL injection.

### ShiftJIS

While looking through a quick summary of Shift-JIS, the following stood out:

>Shift JIS is possible to use in string literals in programming languages such as C, but the 0x5C byte will cause problems when it appears as second byte of a two-byte character, because 0x5C, normally backslash, here ¥, will be interpreted as an escape sequence which will mess up the interpretation.

As a result of the above, and that the MySQL sanitization was being performed manually with a search and replace, we thought that it may have been possible to bypass sanitization using the `¥` character combined with `\"`.

The next question was where to perform this injection. Ideally, we wanted a retrieval operation in order to be able to query the database for arbitrary data while searching for a flag - still assuming, at this stage, that the flag was inside the database.

### WebSockets and `get_answer`

![Well, that was easy.](/assets/article_images/2016/get_answer.png)

Luckily enough, the last operation performed by the application when a survey was complete was to perform a query for a row associated with the user's input answer for `favourite color`.

As a result of this `get_answer` call, we thought that we should be able to simply inject `¥\"` in place of a proper `answer`, followed by some arbitrary SQL, in order to try and locate the flag.

### Fire!

Rather than setting up a new WebSocket from Ruby or Python, we found it easier to just use OWASP Zap to fire requests at an already open WebSocket while the page was loaded. Lazy, yes, but hey! :)

![Fingers crossed](/assets/article_images/2016/sjis-kgo.png)

At this stage, we still weren't sure whether flag was inside of the database, and if it was, where it might be. As luck would have it, however, the first request yielded the flag, and three points for the team.

![Gotcha!](/assets/article_images/2016/sjis-request.png)

The full request sent to the WebSocket which gave us the flag was the following:

```{"type":"get_answer","question":"¥\" OR 1=1;--","answer":""}```

Finally, a big thanks to @BKPCTF for running the CTF! :)
