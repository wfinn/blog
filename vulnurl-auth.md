# Attacking LNURL-auth

Let's talk about LNURL-auth!  
It seems to be the most adapted login mechanism for lightning network apps and one of my targets used it, so I looked at it a bit more. Turns out, it's not as secure as one might assume.  
Don't get me wrong, I couldn't log in to any account or crazy things like that, but today I wanna highlight that it indeed does have some phishing potential due to what *I think* is a small design issue and a deviation from the spec in a well-known implementation of it.

This is kind of a dead project on my end, this post is mostly theoretical and I left a lot unexplored. I got bored of it and thought I write up my thoughts.

## How does LNURL-auth generally work?

It would be ideal if you read [this little explainer](https://fiatjaf.com/e0a35204.html) before you continue, but I'll cover the important things.  
You could also just read [the spec](https://github.com/lnurl/luds/blob/luds/04.md), it's not that long.

When you want to log into a website using LNURL-auth, you get a QR code that you scan with your wallet, which then presents you with some dialog to accept the login.

This QR code contains a LNURL encoded URL, which may look like in the image below.
![[image](https://www.zapread.com/lnauth/signin?tag=login&k1=32-bytes-in-hex)](https://user-images.githubusercontent.com/42862612/197339306-99c5a146-d981-4069-a62c-8b43a94e07b8.png)

I used this lnurl decoder https://lnurl.fiatjaf.com/codec/.

What the wallet does is take the domain and some secret key it already has, and it generates a keypair for this domain.  
Then it signs `k1`, and modifies the URL above to contain the public key and signed data.  
Then the app can verify the signature and use the public key as identifier.

## MitM & Phishing

Okay, let's start with a simple one.  
`k1` is just a bunch of random data, nowhere in the spec does it mention, that it has to be checked that `k1` originated from the server or wasn't used before.  
Let's assume we can replay the login process by resubmitting `k1` and `sig` (you can verify it with burp on your favorite site that uses it).  

Replaying would work. But can we exploit that? We do not have access to `k1` and `sig` of users, even as man in the middle, it is encrypted.  
Well, nowhere in the spec does it say you *have* to use HTTPS :)  
Consider a case where you're MitM but can't access the plaintext traffic to `target.tld` due to TLS.  
Well, you might still be able to phish a user to log in via HTTP.

One could create a QR with a LNURL to `http://target.tld/auth-endpoint?tag=login&...` and show that on `someotherwebsite.tld`, where the user does the usual login flow.  
Usually, this scenario should not lead to a successful account takeover, LNURL-auth should protect against that, but with MitM and using http instead of HTTPS, it can work.

I do not know if any wallet refuses to load HTTP links, haven't explored this too much as it is a bit boring.

## Parser differentials in wallets?

Okay, what if I could create a case where one domain is used for the key derivation, but another is resolved in the network code?  
Maybe some wallet uses a custom parser to get the domain and the HTTP code parses the URL differently.  

![Phoenix wallet LNURL-auth login screen](https://user-images.githubusercontent.com/42862612/197419022-238a947c-83a5-4f9a-9175-a32c2c7b3684.png)

I have only done some minimal testing here, but I'm kind of sure there is a vulnerable wallet somewhere.

## Leaking sig with redirects

If not with the wallet, then with the service!

Consider a redirect like this: `https://legit.tld/redirect/anotherwebsite.com/xyz?x=y`.  
Some websites might have something like that for tracking, of course also a bug like `https://legit.tld////anotherwebsite.com/xyz?x=y` would work.  
What is important is, that the redirect also *forwards URL parameters*.  
Admittedly, this is not that common, but it happens.

This is the core idea I wanted to share, **LNURL-auth + certain kind of open redirect = phishing**.

Of course, you should also consider other GET based leaks, but redirects seem the easiest to find in the real world.  
Together with the fact you can replay this login request forever, it kinda seems like an interesting trick for me.

### Expanding the attack surface

So it looks like one of the most widespread wallets interpreted the spec a bit differently than others. [Phoenix ignores subdomains (but honors eTLDs)](https://github.com/ACINQ/phoenix/blob/b92275fbb324379e274e85125a504c9ce6dafab1/phoenix-shared/src/commonMain/kotlin/fr.acinq.phoenix/managers/LNUrlManager.kt#L255)!  

![Phoenix Wallet code snippet which is likely the reason](https://user-images.githubusercontent.com/42862612/197338604-80af713d-6bc7-427f-919a-65ceebd09631.png)  
`eTldPlusOne` does exactly what it says, this means for a hostname like `www.example.com` it will pick the TLD "com", plus one, which is "example".

This means the keys and signatures for any subdomain are the same when using Phoenix... do you see where this is going yet?  
What if you take over some subdomain on your target, what if it has an applicable open redirect?

Of course, I have talked to Phoenix about this, and they hinted me at a [public discussion](https://github.com/lnurl/luds/issues/55) where they talk about this.  
This is a known fact, they don't seem to think it's an issue. Good news for me tho, as this trick continues to exist for the foreseeable future.  

They noted, and I want to highlight that explicitly, that this code honors effective TLDs, so for `username.github.io` the eTLD would be `github.io`, thus the most well-known shared domains are unaffected.  
But it's still a nice little trick: subdomains are in scope when phishing Phoenix users.

## Final Words

Do not forget you can attack other parts of the LNURL-auth login flow as well. This only covers some LNURL-auth stuff I noticed.  
I also want to clarify that I shared most of this info with the spec author before publishing this post.  
How I'd fix it? Probably just use the full URL (removing the value of k1) for key generation, that would prevent a lot of attacks.  
That would also make LNURL-auth less flexible and kinda disable all current logins, as the keys generated would be different after the update... that's unlikely to happen. I do not think you can compare it to WebAuthn, but I still like the general idea.

I would love for someone else to explore this further or even find some real-world examples.
