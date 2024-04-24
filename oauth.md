# OAuth
My notes management site is using OAuth authentication now, so I can open it to the Internet with peace of mind.

*flag format: 0ops{[0-9a-z]{1,}}

___

## Log leakage

Open the webpage and find the View Notes link on the homepage. Click on it and you will be redirected to the OAuth login interface (the Notes page also requires login). If you have an SSO account, after logging in, you will be prompted that you are not an administrator user.

![](https://s2.loli.net/2024/04/24/UYex27XDGkQENW3.png)

![](https://s2.loli.net/2024/04/24/p6jTuUD8C2NA4Gc.png)


Examining the source code of the web page, both the meta description and the console directed you to look at the head section of the web page.

Hint one is that you can complete this task without an SSO account.

![](https://s2.loli.net/2024/04/24/47JSKWCsGfVxTnv.png)

Hint 2 is the sitemap.xml file in the HTML header. Upon opening it, you will find a code.php file. When accessing it directly, you will be prompted for a code parameter. After giving the code parameter, you will receive a "log saved" prompt, followed by a login failure prompt.

![](https://s2.loli.net/2024/04/24/kT6fisnENS7epyV.png)


Here "log" is bold, corresponding to the hint of the last link in the sitemap.

![](https://s2.loli.net/2024/04/24/1AtQSW628CluPnZ.png)


By accessing the "/log" path, you can see a log of a GET request indicating that the administrator's access record has been recorded.

![](https://s2.loli.net/2024/04/24/ebcHTPd4xmi2WO5.png)


## authorize_code hijack

Since accessing code.php with the code parameter requires a 5 second wait before the server jumps to oauth.php and requests a token from the OAuth server with this authorize_code, we can take advantage of this 5 second time lag by accessing oauth.php with the code parameter of the administrator account before the administrator.

(Actually, the authorization code of this OAuth server is valid for 1 minute, and the code in the log will not be used up after 5 seconds, so we just need to request code.php within one minute after accessing /log, or directly access the path in the log as well)

![](https://s2.loli.net/2024/04/24/XQqhvWYrnZdDpIC.png)

Using the administrator's code, after logging into the administrator's account, we can see the flag format. But the flag needs to get the SSO account name of the administrator which is not displayed on the website. So we have to consider further utilization of admin's authorization code.

![](https://s2.loli.net/2024/04/24/ANMr68PqCGUR9F4.png)


Log in again and find that the authentication URL for OAuth login is `https://jaccount.sjtu.edu.cn/oauth2/authorize?response_type=code&client_id=ZjpxY3dA6fpkp7o4kM0g&redirect_uri=http%3A%2F%2F{hostname}%2Fcode.php&scope=openid`

Retrospecting [OAuth login process](https://datatracker.ietf.org/doc/html/rfc6749), the server needs to obtain the SSO account name of the administrator using user's authorize_code, client_id and client_secret to request token_url to get access_token. The client_id can be obtained from the above authorize_URL, but the client_secret is still unknown.

## client_secret leakage

After logging in with the administrator account, there is a prompt that the "secret" has been underlined. At this point, we can consider the client_secret leak. Searching for the value of client_id on GitHub can find [young1881/SJTUer](https://github.com/young1881/SJTUer/blob/master/django/sjtuers/settings.py#L123), which uses this client_id and leaks the corresponding client_secret.

![](https://s2.loli.net/2024/04/24/Nbmf1ViAKBoWqcX.png)


After obtaining the authorize_code, client_id, and client_secret, we can act as the server and request access_token from the OAuth server to obtain user information. The API for getting a token is the same as the example given in [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749), just replace /authorize with /token at the end of authroize_url.

Send the following request:

POST `https://jaccount.sjtu.edu.cn/oauth2/token`

`grant_type=authorization_code&code=8edfcd24fd074f2faedbb0982fbe74bf&redirect_uri=http%3A%2F%2F{hostname}%2Fcode.php&client_id=ZjpxY3dA6fpkp7o4kM0g&client_secret=CE1FEABAD368510B161F8F0E582CBA6864EAF4137FC18079`

The grant type=authorization_code is specified in RFC 6749. The code is the authorization code obtained from the administrator on the /log page. The redirect_uri needs to be consistent with the redirect_uri in the authorize_URL, and the values obtained can be filled in for client_id and client_secret.

The format of the obtained reply is as follows:

```json
{"access_token":"5f42337796b1b73e71ad9db0cfd82304","refresh_token":"4ad3742daf852219059b386b7c58eb8c","id_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJaanB4WTNkQTZmcGtwN280a00wZyIsImlzcyI6Imh0dHBzOi8vamFjY291bnQuc2p0dS5lZHUuY24vb2F1dGgyLyIsInN1YiI6Im5pY2RhdGEiLCJleHAiOjE3MTA0MzE5MTQsImlhdCI6MTcxMDQzMDExNCwibmFtZSI6Iue9kee7nOS_oeaBr-S4reW_gyIsImNvZGUiOiIiLCJ0eXBlIjoidGVhbSJ9.JpCbW0bP_V_7huFQ2jbOhSfD8nreGFKPBARrfTtbxlw","token_type":"Bearer","expires_in":1799}
```

Among them, id_token is a unique feature of jAccount SSO used in the task, which directly returns the user's information in this step. After decrypting this JWT, the SSO username of the administrator can be obtained in the sub field. Follow the hint after logging into the administrator account on the website, hash ( sha1(sha256(nicdata)) ) to obtain the flag.

```json
{
  "aud": "ZjpxY3dA6fpkp7o4kM0g",
  "iss": "https://jaccount.sjtu.edu.cn/oauth2/",
  "sub": "nicdata",
  "exp": 1710431914,
  "iat": 1710430114,
  "name": "网络信息中心",
  "code": "",
  "type": "team"
}
```

Another approach is to continue following the standard OIDC process. Use the access_token obtained in the previous step to access the user information API. This solution requires consulting the development documentation. Searching for authorize_url, token_url, or logout_url values in a search engine will bring up the [developer documentation](https://developer.sjtu.edu.cn/auth/oidc.html) for this SSO. It provides an example of accessing the API to obtain user information.

Send the following request:

GET `https://api.sjtu.edu.cn/v1/me/profile?access_token=5f42337796b1b73e71ad9db0cfd82304`

Or

POST `https://api.sjtu.edu.cn/v1/me/profile`
Authorization: Bearer 5f42337796b1b73e71ad9db0cfd82304

Where "access_token" or "Authorization: Bearer" is filled in with the obtained access_token.

The format of the obtained reply is as follows:

```json
{"errno":0,"error":"success","total":0,"entities":[{"account":"nicdata","name":"网络信息中心","kind":"canvas.profile","timeZone":0}]}
```

The account field is the SSO username of the administrator.
