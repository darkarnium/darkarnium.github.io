---
layout: post
title: BKP CTF - Wackusensor Write-Up
date: 2017-02-26 00:00:00.000000000 -08:00
author: Peter Adkins
image:
---
Given the quality of the last Boston Key Party (BKP) CTF it wasn't unexpected that there would be some great challenges again this year. Wackusensor certainly fell into that category, providing an interesting target while not being as quite as difficult to solve as some of the other cloud challenges.

This was a great Saturday afternoon challenge, yielding 100 points for our team (`Haxation Without ROPresentation`) after a bit of documentation review and successful exploitation of some fairly common vulnerabilities encountered when using certain PHP functions incorrectly.

### Wackusensor

When first opened in a browser, the Wackusensor challenge landing page mentioned that although 'Acunetix Acusensor' had been deployed to the server it wasn't working as expected.

![O Hai](/assets/article_images/2017/waku-home.png)

The first link in the challenge text links to the Acunetix Acusensor website, and the second to a `.txt` copy of the installed `acu_phpaspect.php` instrumentation script.

### TL;DR: The Solution

```
curl -D - "http://54.200.58.235/index.php?super_secret_parameter_hahaha=php://filter/read=convert.base64-encode/resource=/var/www/html/super_secret_file_containing_the_flag_you_should_read_it.php" \
-H "Acunetix-Aspect: enabled" \
-H "Acunetix-Aspect-Password: 4faa9d4408780ae071ca2708e3f09449"
```

### RTFM

As the provide `acu_phpaspect` script had been 'minified' the first step was to reformat the code into something more readable in order to gain an understanding into what this script would do when called. In addition to this, the [Acunetix Acusensor installation manual](http://www.acunetix.com/support/docs/installing-acusensor/) was consulted in order to understand how this instrumentation _should_ work when properly deployed.

The found documentation mentioned that an installation specific `acu_phpaspect.php` file (generated during setup) should be installed onto the web server, and the `php.ini` configuration file be updated to include this script as part of the `auto_prepend_file` directive. A quick search of the PHP manual for this directive yielded the following:

![Curiouser and curiouser!](/assets/article_images/2017/curious.png)

Given that the documentation indicated that the `acu_phpaspect.php` script would be 'auto-loaded' before any PHP is executed on the server, we were fairly certain that it likely contained a method to extract the flag.

### Keys And Headers

A quick review of the reformatted `acu_phpaspect.txt` indicated that two HTTP Headers would need to be present in a request in order for the instrumentation to execute. This code also made reference to a 32-character hash called `HTTP_ACUNETIX_ASPECT_PASSWORD`; which was automatically read from the end of the same instrumentation script on execution:

```php
$_ENV['_AAS0'] = (isset($_SERVER["HTTP_ACUNETIX_ASPECT"]) && $_SERVER["HTTP_ACUNETIX_ASPECT"] === "enabled");
if ($_ENV['_AAS0']) {
    $_ENV['_AAS0'] = false;
    if (isset($_SERVER["HTTP_ACUNETIX_ASPECT_PASSWORD"])) {
        $_AAS1 = fopen(__FILE__, 'r');
        fseek($_AAS1, -32, SEEK_END);
        $_ENV["_AAS2"] = stream_get_contents($_AAS1, 32);
        unset($_AAS1);
        $_ENV['_AAS0'] = $_SERVER["HTTP_ACUNETIX_ASPECT_PASSWORD"] === $_ENV["_AAS2"];
    }
}
```

In order to confirm our suspicions were correct, the Acunetix Acusensor manual was consulted once again, and the key extracted from the end of the provided `acu_phpaspect.txt`.

![A key? A key!](/assets/article_images/2017/waku-acu_phpaspect_txt.png)

### Twiddling Bits

Now that we had the keys and a basic understanding of how to invoke the installed instrumentation it was time to give it a shot and see what happened. The two HTTP headers which were required to be present in order for the instrumentation to work were as follows:

 * `Acunetix-Aspect-Password`
   * Containing the 32-character hash from the end of `acu_phpaspect.txt`.
 * `Acunetix-Aspect`
   * Containing the string `enabled`.

As luck would have it, when an HTTP request was submitted to the server with the above headers present, the following HTML comments were 'automagically' appended to the response by `acu_phpaspect`:

```
$ curl -D - "http://54.200.58.235/index.php" \
-H "Acunetix-Aspect: enabled" \
-H "Acunetix-Aspect-Password: 4faa9d4408780ae071ca2708e3f09449"

HTTP/1.1 200 OK
...
<!--BKPASPECT:MDAwMDAwMDRQQU5HbjAwMDAwMDAwMDAwMDAwMDBu--><html>
...
<!--BKPASPECT:MDAwMDAwMTBQSFBfRmlsZV9JbmNsdWRlczAwMDAwMDNDc3VwZXJfc2VjcmV0X2ZpbGVfY29udGFpbmluZ190aGVfZmxhZ195b3Vfc2hvdWxkX3JlYWRfaXQucGhwMDAwMDAwMTcvdmFyL3d3dy9odG1sL2luZGV4LnBocDAwMDAwMDBGczAwMDAwMDE1ImluY2x1ZGUiIHdhcyBjYWxsZWQuMDAwMDAwMEFWYXJfQWNjZXNzYTAwMDAwMDAyMDAwMDAwMDNHRVQwMDAwMDAwMXMwMDAwMDAxNy92YXIvd3d3L2h0bWwvaW5kZXgucGhwMDAwMDAwMTJuMDAwMDAwMEFWYXJfQWNjZXNzYTAwMDAwMDAyMDAwMDAwMDNHRVQwMDAwMDAxRHN1cGVyX3NlY3JldF9wYXJhbWV0ZXJfaGFoYWhhMDAwMDAwMTcvdmFyL3d3dy9odG1sL2luZGV4LnBocDAwMDAwMDE0bg==-->
```

### Encoding To Victory?

Given the appended `BKPASPECT` strings looked suspiciously like Base64 encoded data, the value of the larger of the two was piped into `base64 -d` in order to attempt to decode it.

After a small amount of clean-up the following was the result of the Base64 decode:

![Not So Super Secret Squirrel](/assets/article_images/2017/waku-super_secret_squirrel.png)

Given the name of the included PHP script at the top of the decoded `BKPASPECT` string, the first thing we tried was fetching the `super_secret_file_containing_the_flag_you_should_read_it.php` file directly:

![No Dice](/assets/article_images/2017/waku-no_dice.png)

Unfortunately, both the output as well as the `BKPASPECT` from this file did not contain the flag. Boo!

### Not So Secret

In addition to just the `super_secret_file...` in the original request, a reference to an HTTP GET paramater named `super_secret_parameter_hahaha` was also seen.

Given that the flag was supposedly inside of `super_secret_file_containing_the_flag_you_should_read_it.php` - which was not able to be read directly - this additional `secret` parameter was likely the path to getting the flag. In order to determine what this HTTP GET parameter was intended to be used for, `super_secret_parameter_hahaha=true` was appended to the initial request and the request was submitted once again.

![Uh oh.](/assets/article_images/2017/waku-file_get_contents.png)

The output stated, plain as day, that the value of the `super_secret_parameter_hahaha` HTTP GET parameter was being passed directly to a PHP `file_get_contents()` call. In order to test whether things were going to be easy, the value of this parameter in our request was changed to `super_secret_file_containing_the_flag_you_should_read_it.php` and the request resent.

![Of course not.](/assets/article_images/2017/waku-expected.png)

Of course, we didn't actually expect it to be so easy!

At this stage it appered that the value of the `super_secret_parameter_hahaha` parameter was being filtered to restrict loading `php` files, so a different strategy was required in order to retrieve the flag.

### Sideline: The PHP 'Protocol'

> In PHP, a protocol handler named `php://` exists in order to allow for 'accessing various I/O streams.' One of the functions provided by this protocol handler is `filter`: allowing for easy filtering of various other I/O streams directly inside of the URI.
>
> This protocol handler is incredibly useful in a number of cases, especially when looking to bypass 'restricted' input :)

### Give Us The Flag Already!

In an attempt to bypass the filename filter which was blocking the ability to load the file containing the flag a `php://filer` URI was provided in the value of the `super_secret_parameter_hahaha` parameter. This filter was constructed to requested the `/var/www/html/super_secret_file_containing_the_flag_you_should_read_it.php` file be read, Base64 encoded and the result then 'read' by `file_get_contents()`.

![Is it a flag?](/assets/article_images/2017/waku-a_flag_maybe.png)

After a large Base64 encoded block was successfully rendered to the page - outside of the `BKPASPECT` HTML comments - it looked as if this `php://filter` URI had successfully bypassed the filename filters. However, we still needed to Base64 decode this value and confirm that we had the flag.

![It is!](/assets/article_images/2017/waku-a_flag.png)

With that done, we had the flag, an additional 100 points and moved on to some other challenges.

A big thanks to our team, as well as @BKPCTF and co for running the CTF and 'keeping the lights on' all weekend! Great work as always :)
