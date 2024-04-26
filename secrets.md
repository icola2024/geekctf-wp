## Secrets

My notes and secrets are stored in this secret vault. I'm sure no one can get them.

---

The inspiration for the question comes from real attack events, examining file inclusion, Python character capitalization characteristics, and MySQL string comparison characteristics.

When opening a webpage, there is only one login box, and you can find a bunch of strange comments in the webpage source code that you don't know what it is. The console also outputs a string of mysterious numbers. These are actually two hints and are not necessary for solving this problem.

![](https://s2.loli.net/2024/04/26/dzHIfyj4MaOYlTv.png)

![](https://s2.loli.net/2024/04/26/GyJQ7PHCVmS65zf.png)

The mysterious number on the console is ASCII code in octal, and after conversion, the string `Don't you think the color picker is weird?` is obtained, Prompt us to check the color switching function in the upper right corner of the page.

![](https://s2.loli.net/2024/04/26/VcMSJxkRpzPeUmr.png)

> Did you know?
> 
> For such strange encodings, you can use the Magic function of [CyberChef](https://github.com/gchq/CyberChef) for auto-detection.


Switching colors several times and capturing packets, it can be found that when switching colors, the `/setCustomimColor` is requested first, and the response will `Set-Cookie`.

![](https://s2.loli.net/2024/04/26/H1nYO3TrXoMVD8l.png)

The page then retrieves the CSS for the corresponding color from the `/redirectCustomAsset` API, which reads the `asset` value from the cookie and returns the CSS file with the corresponding path.

![](https://s2.loli.net/2024/04/26/QBPWMAmaNeJv6cO.png)

The strange comments in the page source code are Base85-encoded directory structures.

![](https://s2.loli.net/2024/04/26/Znchk9wrW8fYqAB.png)

```plain
.
├── app.py
├── assets
│   ├── css
│   │   ├── pico.amber.min.css
│   │   ├── pico.azure.min.css
│   │   ├── pico.blue.min.css
│   │   ├── pico.cyan.min.css
│   │   ├── pico.fuchsia.min.css
│   │   ├── pico.green.min.css
│   │   ├── pico.grey.min.css
│   │   ├── pico.indigo.min.css
│   │   ├── pico.jade.min.css
│   │   ├── pico.lime.min.css
│   │   ├── pico.orange.min.css
│   │   ├── pico.pink.min.css
│   │   ├── pico.pumpkin.min.css
│   │   ├── pico.purple.min.css
│   │   ├── pico.red.min.css
│   │   ├── pico.sand.min.css
│   │   ├── pico.slate.min.css
│   │   ├── pico.violet.min.css
│   │   ├── pico.yellow.min.css
│   │   └── pico.zinc.min.css
│   └── js
│       ├── color-picker.js
│       ├── home.js
│       ├── jquery-3.7.1.min.js
│       └── login.js
├── gunicorn_conf.py
├── populate.py
├── requirements.txt
└── templates
    ├── base.html
    ├── index.html
    └── login.html
```

Trying to modify the `asset` in the cookie to read out the other files in the directory returned `Hacker!`, guessing it might be doing a check on the path string header.

![](https://s2.loli.net/2024/04/26/kFvrOM4TVc1pInx.png)

Trying to bypass the check with `../`, and found that the read was successful, so we are able to drag down the entire site's source code.

![](https://s2.loli.net/2024/04/26/nNVbXlqjm8JCMFy.png)

The main logic of the site is in `app.py`, so let's start by looking at the code for the login section.

```python
@app.route("/login", methods=["GET", "POST"])
def login():
    if session.get("logged_in"):
        return redirect("/")

    def isEqual(a, b):
        return a.lower() != b.lower() and a.upper() == b.upper()

    if request.method == "GET":
        return render_template("login.html")
    username = request.form.get("username", "")
    password = request.form.get("password", "")
    if isEqual(username, "alice") and isEqual(password, "start2024"):
        session["logged_in"] = True
        session["role"] = "user"
        return redirect("/")
    elif username == "admin" and password == os.urandom(128).hex():
        session["logged_in"] = True
        session["role"] = "admin"
        return redirect("/")
    else:
        return render_template("login.html", error="Invalid username or password.")
```

Obviously logging in as `admin` is not possible, so the only other possibility is to log in as `alice`. But the logic of `alice` username and password is a bit strange, `isEqual()` requires that the two strings are different in lowercase but the same in uppercase, which at first glance doesn't seem possible. But anyway, there aren't too many Unicode characters, so let's enumerate them all and see, and we found 4 of them.

```python
import string

for i in range(0, 0x10FFFF):
    c = chr(i)
    if c.upper() in string.ascii_uppercase and c.lower() not in string.ascii_lowercase:
        print(c, c.upper())
```

```plain
ı I
ſ S
ﬅ ST
ﬆ ST
```

So using `alıce` as the username and `ſtart2024` or `ﬆart2024` or `ﬅart2024` as the password will pass this authentication and log in as `alice`. But since we are not `admin` right now, we can only see `notes`.

![](https://s2.loli.net/2024/04/26/KhjfZC5OgkpQcLN.png)

The next goal is to gain access to `secrets`, so let's see how access control is implemented.

```python
type = request.args.get("type", "notes").strip()
if ("secrets" in type.lower() or "SECRETS" in type.upper()) and session.get(
    "role"
) != "admin":
    return render_template(
        "index.html",
        notes=[],
        error="You are not admin. Only admin can view secre<u>ts</u>.",
    )
```

The logic here is that if the current user is not `admin`, then the incoming `type` is checked and access is denied if `type` contains `secrets` in lowercase or `SECRETS` in uppercase. At first glance, it may seem like there are no flaws, but the filtering mechanism of this blacklist is worth questioning whether there is a way to bypass it.

After carefully reading the code again, we found two seemingly useless assertions, telling us that the Character Set in the database is `utf8mb4` and the Collation is `utf8mb4_unicode_ci`.

```python
assert character_set_database[0] == "utf8mb4"
assert collation_database[0] == "utf8mb4_unicode_ci"
```
So what are Character Set and Collation? Check the official MySQL documentation for an explanation.

> A character set is a set of symbols and encodings. A collation is a set of rules for comparing characters in a character set.

We note that Collation determines the rules for comparing characters in a Character Set, and that MySQL compares two strings based on their **Weight**, which is determined by Collation. We simply use the `utf8mb4_unicode_ci` Collation with the same Weight as `secrets`. There are many strings that meet this condition, and the fact that the `ts` of `secrets` is underlined suggests a solution, some of which are listed below:

```plain
secreʦ
Śecrets
secre%00ts
secréts
```

Of course, if you don't know this, you can set up the exact same environment locally, with the same Character Set and Collation, and enumerate the Unicode characters in the same way as before, and you can also find strings that can bypass the check.

Finally, the real-life incident behind the task. in December 2023, the OKX exchange was attacked due to an improperly set Collation. The attacker spoofed the database with `saʦ` and managed to impersonate `sats` inscription tokens in the search results, so users with poor eyesight fell for the scam and were played for a sucker by the hackers.

![](https://s2.loli.net/2024/04/26/Rxtk6VG9APJYaMf.png)

In fact, this is not the fault of the database software, but the failure to use the correct Collation when comparing strings, which can be circumvented by using `utf8mb4_unicode_bin`.
