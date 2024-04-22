# ECommerce

David, from a multinational e-commerce company, is developing a brand new front-end of the mall system for the company. But David was so lazy that he only finished the login screen in a whole week! Moreover, due to a configuration error, the new website was accidentally exposed to the public web.

http://chall.geekctf.geekcon.top:40472/

***

This is a simple penetration task. You need to access the backend through a seemingly harmless login page exposed on the public network, and find the four scattered flag parts in the system. It involves many unconventional web knowledge points, including Graphql API, OSINT, front-end project deployment, source code leakage, etc.

## Landing
Open the webpage given in the task and come to a classic login page.

![](https://s2.loli.net/2024/04/22/sWAOtTF47Z9Xoc8.png)

Based on information such as "DeepShop," "dashboard," and "ECommerce," it is speculated that this is the backend of a shopping mall system.

Without further ado, enable packet capture and try logging in using admin/admin.

```http
POST http://chall.geekctf.geekcon.top:40472/api/ HTTP/1.1
Host: chall.geekctf.geekcon.top:40472
Connection: keep-alive
Content-Length: 172
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 Edg/124.0.0.0
Content-Type: application/json
Accept: */*
Origin: http://chall.geekctf.geekcon.top:40472
Referer: http://chall.geekctf.geekcon.top:40472/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7

{"query":"mutation MyMutation {\r\n  tokenCreate(email: \"admin\", password: \"admin\") {\r\n    errors {\r\n      code\r\n      message\r\n    }\r\n    token\r\n  }\r\n}"}
```

Response：
```http
HTTP/1.1 200 OK
Server: nginx/1.25.4
Date: Sun, 21 Apr 2024 11:05:13 GMT
Content-Type: application/json
Content-Length: 211
Connection: keep-alive
referrer-policy: same-origin
x-content-type-options: nosniff
access-control-allow-credentials: true
access-control-allow-origin: http://chall.geekctf.geekcon.top:40472
vary: Origin

{"data": {"tokenCreate": {"errors": [{"code": "INVALID_CREDENTIALS", "message": "Please, enter valid credentials"}], "token": null}}, "extensions": {"cost": {"requestedQueryCost": 0, "maximumAvailable": 50000}}}
```

There are at least two things that are suspicious about this request:
- Common RESTful APIs specify resource names in the URL (e.g. `/api/books`, `/api/v2/user`, etc.), but this request uses `/api` directly, so it's possible to guess that this endpoint is capable of handling all of the system's required requests.
- The Request Body is also unusual (it should logically be `{"email": "admin", "password": "admin"}`), and here it uses a very special format.

In fact, a quick lookup (e.g., searching with the keyword "mutation API") or asking GPT will tell you that this API is a Graphql API.

## Set-off
It is not difficult to find some useful information by searching with the keyword "Graphql ctf", such as
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection
- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/graphql#introduction

After reading the articles, you can learn that using 'IntrospectionQuery' can obtain all the schemas. Try using Postman:

![](https://s2.loli.net/2024/04/22/3RgrXwpLqU1umsa.png)

The whole 2+MB of JSON is too much to read, so we switch to the playground and I use [graphiql](https://github.com/graphql/graphiql/blob/main/examples/graphiql-cdn/index.html), replace the address on line 68 with `http://chall.geekctf.geekcon.top:40472/api/`. By the way, delete the custom header in its example (otherwise there are CORS issues across domains) and run it on nginx/live server.

|     |     |
| --- | --- |
| ![](https://s2.loli.net/2024/04/22/An3GVuJqOtX1Zzw.png) | ![](https://s2.loli.net/2024/04/22/mYs9yuq3naZiBHG.png) |

Both Docs and APIs are now available.

## Explore
The first step in penetration is to figure out the current situation:
- Who am I?
- Where am I?
- What can I do?

It's easy to locate a "me" API in the Playground, and check the following fields to request it:

![](https://s2.loli.net/2024/04/22/QgxpI3oRwZSEtcN.png)

Why is it null? Because we haven't logged in ...... at all.

Ahem, let's move on to the next question. After all this time, we still don't know what kind of system this is (but we can guess that it's probably very complex because there are so many APIs), so we tried a bit and found that there's a "shop" API, so let's request it.

![](https://s2.loli.net/2024/04/22/yOtXglrKixCZT1q.png)

Some of the fields require permissions, we remove these fields.

![](https://s2.loli.net/2024/04/22/2uIWzrxobEj5AH9.png)

Found out it's a system called "Saleor", Github search found the target (as expected, since the task setter couldn't have written such a complex system by himself ...... )

![](https://s2.loli.net/2024/04/22/6y3VibaTcoKYFBL.png)

A brief look at its architecture reveals a system that separates front-end and back-end.

![](https://s2.loli.net/2024/04/22/nvl48yqMuoAp7zS.png)

So clone its front-end project (https://github.com/saleor/storefront), put the API address in `.env`, and `npm run dev` to run it.

```
NEXT_PUBLIC_SALEOR_API_URL=http://chall.geekctf.geekcon.top:40472/api/
```

Enter the homepage of the mall.

![](https://s2.loli.net/2024/04/22/AbPZumJsKTBWGQ4.png)

After searching, it's not difficult to find two hints on the About page:

![](https://s2.loli.net/2024/04/22/hPXYyQ9imuveE7F.png)

```
hint 1: flag part 1 is not in the db, but in a description of a model.

hint 2: - hey bro, where's The Dash Cushion? - Just search for it.  
```

- For hint 1, the flag is not in the database but in the model description. Recalling the Graphql schema mentioned earlier, search for the flag in the response obtained in Postman to obtain the flag part 1: `ZmxhZ3s5ckBQSH`
![](https://s2.loli.net/2024/04/22/dP9DMnbaOT7ysv2.png)
- For hint 2, searching for "The Dash Cushion" in the search box on the frontend didn't seem to turn up anything special, so we switched to the playground and found flag part 2: `ExX0BQIV8zWH`
```
query MyQuery {
  products(
    search: "The Dash Cushion",
    channel: "default-channel", 
    first: 10
  ) {
    edges {
      node {
        availableForPurchase
        availableForPurchaseAt
        channel
        chargeTaxes
        created
        description
        descriptionJson
        id
        externalReference
        isAvailableForPurchase
        name
        rating
        seoTitle
        slug
        seoDescription
        updatedAt
      }
    }
  }
}
```
![](https://s2.loli.net/2024/04/22/izH2LZckjURtlsW.png)

## Breakout
Continuing to try in the playground, it was found that many APIs require permissions, and there doesn't seem to be any more information in the publicly available APIs.

![](https://s2.loli.net/2024/04/22/rW5e2TaDHgoxtIy.png)

According to the basic process of penetration, it is time to find a way to get admin right. But this open-source project with 20.1k star doesn't seem to have any vulnerabilities either. So we went back to the original webpage and flipped through the source code.

![](https://s2.loli.net/2024/04/22/Iz8rnfOF91PovgG.png)

It is not difficult to find that the site has source mapping which results in source code leakage, in pages folder there is only an index.tsx (in line with the background of David is very lazy), and in this file, a line of comments gives David's e-mail address: `david@deepshop.co`

Guess password '123456', found successful login:

![](https://s2.loli.net/2024/04/22/A2EWbhDIsfxjTPk.png)

But there seems to be no access token. Switch to playground anyway:
```
mutation MyMutation {
  tokenCreate(email: "david@deepshop.co", password: "123456") {
    csrfToken
    refreshToken
    token
    user {
      account
      isStaff
      userPermissions {
        code
        name
      }
    }
  }
}
```

Got the token (it's a JWT), as well as knowing that David is staff and has a bunch of permissions.

![](https://s2.loli.net/2024/04/22/XEcfyP3GKxQWDJ6.png)

Put the token in the Authorization header and you're ready now!

![](https://s2.loli.net/2024/04/22/U9qN4BGyDZd6mOk.png)

## Arrival
In fact, in addition to storefront, there is a dashboard front-end project (https://github.com/saleor/saleor-dashboard). Lets us run (note checkout version 3.18) and log in directly as David.

![](https://s2.loli.net/2024/04/22/hs9lq6bP2BFVd7e.png)

This makes finding the flag much more friendly than using the API. It's easy to see that the first customer's details have the hint.

![](https://s2.loli.net/2024/04/22/7k9z5oQUIiVSZL4.png)

```
hint 3: maybe flag part 3 was sent to 076 Richard Park Apt. 040
hint 4: what's “GraphQL API” means in Chinese？
```

Following tip 3, find #4 in Orders, Derek Clark's order, and get flag part 3 in Order History.：AwNWU1X0VcL

![](https://s2.loli.net/2024/04/22/pw38M7eVKg4GiQR.png)

Following tip 4, find Translations - Chinese - Menu Items - GraphQL API, get flag part 4：zNyWStoSU45fQ==

![](https://s2.loli.net/2024/04/22/NIeXyaQJAc16sGo.png)

Finally, concatenating the string and base64 decoding yields the flag.
```
flag{9r@PHq1_@P!_3Xp05e5_E\/3rY+hIN9}
```

## Conclusion & Reflection
Web knowledge points are numerous and tedious. This time, I didn't have a particularly good idea, but I remembered that I happened to be doing some development with Saleor, so I thought about whether I could use it to come up with some different web tasks. I noticed that all the other web tasks were classic knowledge points, so I hit it off with a comprehensive penetration!

Saleor itself is a very good system with excellent architecture. In terms of API, Graphql is a very new specification. When consulting materials, I also found that Graphql had appeared in previous CTF competitions and had the elements to set questions.

In fact, in the end, you will find that after entering the dashboard, everything becomes very simple, you just need to click the mouse to rummage around to find the flag. But before you reach this step, you may encounter a lot of obstacles, such as the use of Graphql and endless errors of Saleor projects  (both backend and frontend).

There's one more little tidbit. Originally, this David-written (what David-written? I wrote it) login page was a static html, with the email placed directly in the comments. Then I thought it was a bit too easy, so I rewrote it in ReactJS, and manually enabled the source mapping when I exported it, and the mapping generated this way is displayed as "\_N\_E" in the list of developer tools in the browser (if you `npm run dev` directly, it is usually displayed as "webpack://"), which is not quite the same as the normal one. It turns out that this change seems to have stopped many people, who automatically ignores "\_N\_E". (I am using the default configuration, and don't know why NextJS uses this string by default.)

Well, all in all, it's still important not to forget the soul of penetration testing —— information gathering, whether it's getting David's email or learning that the back-end system is Saleor.
