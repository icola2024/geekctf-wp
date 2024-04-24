# Next GPT

They say CTF held after year 3202 must contain a challenge of GPT.
Access Code: 20244202

---

Open the webpage, enter the access password, and enter the main page.

![](https://s2.loli.net/2024/04/24/qtokuIciHjsdNEX.png)

Searching for vulnerabilities in NextChat, we found CVE-2023-49785, which affects version ≤ v2.11.2. Our website's version happens to be 2.11.2 and can be directly exploited.

Trying to talk to the GPT, we find that the reply is a couple of fixed texts. There is some probability of getting a hint:“I did tell GPT the flag, but I made an IP control of this api, so I'm the only person that can request it locally.”

Try asking the flag and get “I'm sorry, I cannot assist with this request.”

![](https://s2.loli.net/2024/04/24/Sc8fMj6l3hrgVo4.png)

![](https://s2.loli.net/2024/04/24/z7Pe9tcEwJqCZXv.png)

We can know that a local IP address is required to access the API. Use the SSRF vulnerability in CVE-2023-49785.

Attempt to inquire about the flag and capture the packet to change the request /api/openai/v1/chat/completions when communicating with the GPT to /api/cors/http/localhost/api/openai/v1/chat/completions. By exploiting the SSRF vulnerability and connecting through a local IP, the flag can be obtained.

![](https://s2.loli.net/2024/04/24/VtklCRfP8LyxnGe.png)

This backend did not use LLM, but instead simulated a completion interface, as can be seen from its response:

![](https://s2.loli.net/2024/04/24/cXKQFW8rDJgkdbT.png)

The authors wanted the contestants to free their minds from the large number of CTF tasks containing GPTs, and used a Web task to remind the contestants that GPT tasks are not necessarily Misc, but may be Web tasks.

![](https://s2.loli.net/2024/04/24/u7eYjrK5q1oLJpb.png)

Additionally, since NextChat only specifies the deployment method using Docker in the documentation, the actual access required through SSRF is to the local 3000 port. Actually our backend is determined by whether the path is /api/cors/http/ or not, so that both the localhost:3000 and the localhost form can obtain the flag.