---
layout: post
title: BKP CTF - Bug Bounty (Suffolk Downs)
date: 2016-03-06 00:00:00.000000000 -08:00
author: Peter Adkins
image: 
---
Over the last two days I've been participating in the Boston Key Party (BKP) CTF with a group ephemerally known as 'Fear Of A Whitehat Planet'. In the end, we didn't do *too* badly - with all of the web challenges, a couple of crypto, and only one of the pwn challenges complete - but better luck next time, eh? 

The third, and final, web application that we worked on was 'Bug Bounty' ('Suffolk Downs' on the CTF console). This was an interesting challenge which has us stumped for some time. 

### Bug Bounty

![Let's do this](/assets/article_images/2016/bug_bounty.png)

When first loaded, the 'Bug Bounty' challenge was seen to be exactly as it said on the tin, a Bug Bounty submission system.

### The Flow.

After a bit of review, the flow was found to be as follows:

1. User creates an account.
2. User submits a bug.
  + User provided with a SHA1 hash for tracking.
  + User is requested to solve a captcha.
  + New bug is marked with a status of `Pending review`.
3. User solves a captcha.
  + The system marks the bug as `Seen` after completion.

However, now that we believed that we had captured the 'standard flow' of the application, it was time to try and see whether we could affect the flow by tampering with any and all request data.

### The Obvious?

At first, I had thought that this might be another challenge which could be solved with SQL Injection or directory traversal. However, after a number of attempts, it started to look less likely that this was the case. Everything appeared to reply with a simple `Fail` message when omitted, malformed, or intentionally tampered with.

As for directory traversal, this was not only incorrect, but also had me burn a number of hours 'down the rabbit hole'. I have the feeling now, after the fact, that this was indeed a Red Herring added by the challenge designer :)

### CSP.

To quote [H5SC Minichallenge 3](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it's-CSP!%22), "Sh*t, it's CSP!":

```content-security-policy:"default-src 'none'; connect-src 'self';  frame-src 'self'; script-src 52.87.183.104:5000/dist/js/ 'sha256-KcMxZjpVxhUhzZiwuZ82bc0vAhYbUJsxyCXODP5ulto=' 'sha256-u++5+hMvnsKeoBWohJxxO3U9yHQHZU+2damUA6wnikQ=' 'sha256-zArnh0kTjtEOVDnamfOrI8qSpoiZbXttc6LzqNno8MM=' 'sha256-3PB3EBmojhuJg8mStgxkyy3OEJYJ73ruOF7nRScYnxk=' 'sha256-bk9UfcsBy+DUFULLU6uX/sJa0q7O7B8Aal2VVl43aDs='; font-src 52.87.183.104:5000/dist/fonts/ fonts.gstatic.com; style-src 52.87.183.104:5000/dist/css/ fonts.googleapis.com; img-src 'self';"```

It was noticed quite quickly during the initial investigation that there was a very strict CSP (Content Security Policy) applied to all responses from the application. Given that this was present, and that a submitted bug changed status to 'Seen' after a captcha was solved, there was some thought that the intended solution involved a CSP bypass.

However, for 'funsies', I thought I'd give some very basic XXS a shot; just to be sure that the CSP was being applied and working as expected.

![It's worth a shot, right?](/assets/article_images/2016/have-some-javascript.png)

...Not surprisingly, CSP did exactly as it was configured:

!["Yeah? Nah."](/assets/article_images/2016/hah-no.png)

At this stage, we started to look for scripts that we could use for a CSP bypass.

### The Rabbit Hole Begins...

The rabbit hole started when I noticed that the page which renders a captcha on bug submission had an `ng-app` attribute set on the root `html` element.

![Angular, eh?](/assets/article_images/2016/beginning-the-rabbithole.png)

As there were a number of third-party Javascript files present in the `/dist/js/` directory on the web-server - included by the rest of the application - I wondered whether there was an AngularJS application lurking around. Perhaps there was also an administrative interface for this Bug Bounty system?

Although directory indexes were disabled, a lot of the time these files tend to be deployed with a fairly predictable naming convention:

* angular.min.js
  * A minified AngularJS.
* app.js
  * The AngularJS application itself.

Using the common names above, we found that there was was both a minified copy of AngularJS inside the `/dist/js/` directory, as well as the scaffolding for an AngularJS application.

![Ah ha!](/assets/article_images/2016/rabbithole-2.png)

From here, I spent a number of hours trying various CSP bypass tricks with AngularJS in order to avoid being shot-down by the CSP (as in-line Javascript was being quashed by the CSP, listed above).

Unfortunately, all of my attempts culminated the following:

![Effectively, "fuck you".](/assets/article_images/2016/effectively-fuck-you.png)

### Enter META

After spending a bit of time reviewing CSP documentation and reported bypasses, we found that a `META` refresh tag has been demonstrated to be able to be used to hard-redirect to another page without CSP balking. Although this isn't a CSP bypass, we thought that we might be able to use this to our advantage if the system which was marking submissions as `Seen` was leaking any information on redirection (referrer headers, etc).

As a result, we knocked together a one-liner which attempts to immediately redirect the page to an [HTTP request bin](http://www.requestb.in):

![A long shot.](/assets/article_images/2016/a-long-shot.png)

Once submitted, we accessed the submission via `show_report` - which loads the submission into an `iframe` on the page - only to find that the `META` refresh was also being blocked by CSP.

!["...And I would have gotten away with it too if it wasn't for your darn CSP."](/assets/article_images/2016/no-dice.png)

At this stage we were at a loss. However, as one final shot at the moon, we thought we should complete the captcha for this submission anyway; just in case the system marking these reports as `Seen` was rendering reports in a different manner.

![Awww Yiss!](/assets/article_images/2016/flaggy-flag.png)

...To our surprise, not only did the `META` refresh fire after completing the captcha, but the HTTP `User-Agent` of the request was set to the flag!

With the flag captured, we claimed an additional three points, and marked off the last of the web challenges in the BKP CTF.

Once again, a big thanks to @BKPCTF for running the CTF! :)
