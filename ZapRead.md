# Pwning Bitcoin based Social Media

Today I want to tell the story about how I hacked a BTC based social media site called ZapRead.  
The bugs I'll show aren't new things or complex, but the potential impact is huge and they were fun to find.  
I have to smile when I think about these days, so I want to tell the story.


The concept of [https://zapread.com](https://zapread.com) is simple: imagine Reddit, but you up- and downvote with satoshi (parts of bitcoin).  
That way content creators directly earn money and ZapRead gets funded by taking a cut of that.

ZapRead is based on the Lightning Network, the second layer of Bitcoin allowing for small & instant off-chain payments with extremely low fees.
ZapRead essentially acts as [custodial LN wallet](https://cryptodefinitions.com/dictionary/custodial-wallet/), you send some satoshi there and vote with them.  
You can also use your account for BTC LN payments, so it really is a wallet, you do not buy internet points.

## "Do you do site penetration testing? Just asking for a friend"

This story begins over 2 years ago, when a friend introduced me to ZapRead.  
ZapRead is an open-source project by a single person, and it's still in beta at the time of writing.  
Keep that in mind when we look at the vulnerabilities.

One day I shared some hacking related things on ZapRead and Zelgada, the creator of ZapRead, commented:  
"Very interesting!  Do you do site penetration testing?  Just asking for a friend 😉"

We chatted a little and I began hacking ZapRead.

## XSS == $$$

Being a wallet and a relatively huge social media website makes for an interesting target.  
Imagine I could send a chat message to someone with a little javascript that just sends me their money!

You see, XSS can be quite dangerous for ZapRead.  
And since it's so big and untested, I found a couple of those, but this is one of the cooler ones.

ZapRead uses WebSockets for some real-time features, like displaying a popup when someone you follow makes a post or you get a chat message.

On the realtime.zapread.com subdomain, there is an endpoint to send such a message to a person.  
I have actually no idea why this feature is exposed, as there are specific endpoints for chat messages, posting etc. which trigger these alerts/messages internally.

The http endpoint allowed you to send a XSS payload to a single user over WebSockets.

```json
{
  "toUserId":"VICTIM-UUID",
  "content":"<img src onerror=stealAllMoney()>",
  "reason":"popup title"
}
```

![image](https://user-images.githubusercontent.com/42862612/151078131-48f94cdd-58f3-479b-a65d-e645c11aeef9.png)


The obvious fix is to sanitize the html like in every other endpoint, but that's not the interesting part here.  
The cool thing about this bug is that it allows me to be super stealthy.  
There are no chat messages or posts or whatever with XSS payloads the admin could use to prove a hack.  
I'm not sure if there are even logs for this real-time API.

You could have just monitored ZapRead and attack everyone you think has some money when they appear online.

Then it's just about writing a payload that tips you all the victim's money from ZapRead.

Over enough time, spamming the payload to some unsuspecting users, I think you could keep this hidden and get some money.  
To get a bit more time one could even use the XSS to make it look like the user still has their funds.

## Stealing ALL FUNDS at once

Let's be honest, XSS is *boring* compared to just stealing all the money at once!  
ZapRead is backed by a single BTC LN node with all user funds,  
so if there is a bug in ZapRead that gives you more money than you should get, you could steal everyone's money.

I found multiple ways to do exactly that, namely a debug backdoor that somehow slipped into production and spending a negative amount of money.
But "Stealing Money with Anonymous Tips 2.0" was the best one, here's the report I sent:

> A usual request to /Manage/TipUser looks like the following:  
> {"id":1,"amount":100,"tx":null}
>
> Let's take a look at ManageController.cs:449, the code for anonymous tips.
> To reach it we need to meet tx != null. But what is tx?
>
> var vtx = await db.LightningTransactions.FirstOrDefaultAsync(txn => txn.Id == tx);
>
> Seems like it is the id of a transaction. txn.Id is an Int32.
>
> if (vtx == null || vtx.IsSpent == true)
> {
>     Response.StatusCode = (int)HttpStatusCode.Forbidden;
>     return Json(new { Result = "Failure", Message = "Transaction not found" });
> }
>
> So we need to find a transaction where vtx.IsSpent == false.
> If we pass this check, we will get the tip!
>
> receiver.Funds.Balance += amount.Value;
>
> amount.Value is our amount parameter from the request, so we can steal an arbitrary amount of BTC.
>
> Steps to Reproduce:
> 1. Intercept a request to /Manage/TipUser
> 2. Guess a valid txn.Id.
> 3. Extract the money from ZapRead

I exploited a feature for people without account to tip content creators.  
By brute force using Intruder, I found some old transaction IDs that were already paid but not spent.
And because ZapRead didn't verify the amount of the transaction, I could just give myself an arbitrary amount of money.  
To this day I'm not quite sure if this only worked for some old debug transactions, or if I could have used my own transaction while it is IsSpent = false somehow.

ZapRead fixed these money draining bugs incredibly fast usually.

## Malicious Attackers hacked ZapRead!

Yes you read it right, there was actually a real attack and the attackers were able to steal some funds!  
They abused a race condition when depositing/withdrawing BTC, you can read about it [here](https://www.zapread.com/Post/Detail/6599/payments-temporarily-disabled/).  
I don't know all the details but it seems like some of their deposits / withdraws were paid out mutliple times when spamming such requests.

Due to the nature of such race conditions the attackers generated lots of noise and Zelgada was able to detect it before they drained all the funds.

The sad thing is, I already knew about race conditions back then, but didn't test for them on ZapRead yet.  
I kinda felt like a failure, I didn't find this before the bad guys did...

In the end I think I helped protect ZapRead though, because stealing all money at once would have been too fast to detect and stop in time.
It could have been worse, but I felt responsible for ZapRead as I've spend countless hours on it at that point.

## Bypassing XSS detection

Here's a little #bugbountytip for those that don't know it yet.

ASP.NET has some default checks for XSS.  
In usual query or body params, it detects brackets (<>), which blocks most XSSes here.

On usual POST forms I got "potentially dangerous Request.Form value[s]" when testing for XSS.  
Switching from application/x-www-form-urlencoded to application/json completely bypasses that check.  
`param=<xss>` gets blocked, but `{"param":"<xss>"}` doesn't.

Most endpoints on ZapRead require a CSRF token though, and that doesn't get read from json, so this trick not super useful here.

## PrivEsc

I wanted to skip this one. For one, because it's not that interesting technically, but more importantly, because I'm a little ashamed...  
Check the image below.

![image](https://user-images.githubusercontent.com/42862612/151078197-5f747c27-4f23-4e2b-8b6a-945c46798ed7.png)

As you can see, I could just request an Administrator API key.  
The thing is, back when I found this I thought it didn't work.  
I checked this on /Admin and one or two endpoints, but I just got errors, so I ignored it and worked on money related features.  
*Months* later I revisited the API key stuff and just tried it again on some random Admin endpoint and it worked!  
Turns out it doesn't work on all Admin endpoints.  
I'll never sit on an Administrator API key for months again... I shouldn't have given up after a couple endpoints.

## Final Words

I want to give huge shout-outs to Zelgada!  
Zelgada does this all alone and was very interested in my reports, asked questions and fixed most bugs quite fast.  
It's a pleasure reporting bugs to ZapRead and I really hope they keep this attitude towards security once ZapRead takes off.

---

https://zapread.com https://github.com/Horndev/zapread.com
