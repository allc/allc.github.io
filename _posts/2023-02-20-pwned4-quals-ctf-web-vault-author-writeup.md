---
layout: post
title: pwnEd4 Quals CTF Vault (Web) Author Writeup
tags: [ctf, ctf writeup, ctf web, ctf neural network]
---

Challenge code and is available at <https://github.com/allc/Vault-CTF-Chall>.

This is my writeup to the web challenge I created for [pwnEd4](https://pwned.sigint.mx/) Quals CTF organised by University of Edinburgh's cyber security society [SIGINT](https://sigint.mx/). The online CTF was on 18th and 19th for 32 hours. With over 30 teams submitted at least one flag, only 2 teams successfully solved this web challenge.

This challenge exploits improper use of OAuth2 for authentication, more specifically with Implicit Flow. It is inspired by [a vulnerability I made in my own project](https://github.com/allc/CTF-Archive-App/commit/e69dee6244ae723780845d8e54e71b82720fafc8), as well as similar issues found in the wild (in my opinion this can partly attribute to developer docs failing to make it clear about the danger of using OAuth2 Implicit Grant).

To understand this challenge, it is helpful to know about both OAuth2 Authorization Code Grant and Implicit Grant, and here are some resources: <https://developer.okta.com/blog/2018/04/10/oauth-authorization-code-grant-type> and <https://developer.okta.com/blog/2018/05/24/what-is-the-oauth2-implicit-grant-type>.

## The Challenge

The challenges has a "Vault" app, where the flag is stored in.

![Vault](/assets/image/pwned4-quals-ctf-web-vault-author-writeup/vault.png)

Clicking on "View flag", it redirects to the auth server to login. Logging in with a newly created account, it redirects back to the Vault app's flag page, however, it says do not have permission to view the flag.

In the auth service, any registered user can create apps.

![App config page in auth service](/assets/image/pwned4-quals-ctf-web-vault-author-writeup/app.png)

When "publishing" the app, the admin bot will try to authorize the app, resulting in a request to the redirect URL with the authorization code.

## The Solve

Substituting the code with the one in admin bot's request to login to Vault would not work, as the OAuth2 server will be able to tell the code is not for Vault thus not valid, when Vault exchanges token using the code along with its client ID and secret.

The solution to this challenge is to exchange the authorization code for access token with the valid client ID and client secret at auth service's `/oauth2/token` endpoint, and substitute the token into Vault's implicit flow callback.

There are two hints in the challenge that the Vault app supports implicit flow. One is on the auth service config page, where it can generates OAuth2 URL for an app. In the response type selection, there is a "token" option (though it's on the auth service, it's in a CTF challenge, it must be relevant :P). Replace "response_type" from `code` to `token` in the login page and proceed the authorisation normally will confirm Vault supports implicit flow. Another is on the screen confirming authorising Vault app, click "Cancel", the callback page will render Javascript tries to use implicit flow.

![App config page in auth service](/assets/image/pwned4-quals-ctf-web-vault-author-writeup/callback-source.png)

To solve the challenge less painfully, there are some details. One is that the access code can only be used to exchange for token once, within valid time. Another is "state" only stay valid before sent to callback endpoint in Vault. To generate a new valid state, click on "Login" in Vault again, and the state can only be used with the session cookie.

## Conclusion

OAuth2 is for authorisation, and can be dangerous used for authentication without additional checks. Implicit Grant should be avoided. Developer docs should make clear of security implications.
