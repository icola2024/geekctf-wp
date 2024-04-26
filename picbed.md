## PicBed

PicBed is an elegant image hosting service which uses webp\_server\_go to serve your JPG/PNG/BMP/SVGs as WebP/AVIF format with compression, on-the-fly.

P.S. It is recommended to test your exploit locally before creating an online instance.

---

I wanted to raise the level of difficulty a little bit, so I took a very cockamamie vulnerability in an open source project and came up with this question, which examines HTTP request smuggling and Go language code auditing.

Inspecting at the source code in the zip package, we found that the front-end is written in Flask, which is responsible for page display and image upload, and the back-end uses an open-source project [`webp_server_go`](https://github.com/webp-sh/webp_server_go), which is responsible for returning the original image or WebP format image according to the `Accept` request header passed in by the user. Our goal is to get `flag.png` in the container root directory.

[`webp_server_go`](https://github.com/webp-sh/webp_server_go) loads images from `/opt/pics` by default. Obviously, we need to uncover a directory traversal-like vulnerability in the [`webp_server_go`](https://github.com/webp-sh/webp_server_go) project and exploit it. Searching for the `webp_server_go path traversal` keyword, it's easy to see that the project used to have a CVE-2021-46104, but it's fixed in version 0.11.1, the version used in this question, so it doesn't seem to be of much use.

But let's take a look at how CVE-2021-46104 was fixed. Going through the Issues and PRs for the project, we found that the PRs involved in fixing this vulnerability are [#93](https://github.com/webp-sh/webp_server_go/pull/93) and [#103](https://github.com/webp-sh/webp_server_go/pull/103). Further reading of the code changes in these two PRs reveals the following lines of code at the core.

![](https://s2.loli.net/2024/04/26/OJ7puGVcWgRvqLd.png)

From the code, it looks like the developer is trying to avoid directory traversal by eliminating `../` in `reqURI` via `path.Clean()` to avoid directory traversal. A quick look at the [official documentation](https://pkg.go.dev/path#Clean) shows that the `path.Clean()` function returns the shortest pathname equivalent to the argument through purely lexical processing. This doesn't seem to be a problem at first glance, even if `reqURI` has more than one `../`, the elimination seems to end up just going back to `/`, which is then spliced with `config.ImgPath`, which certainly doesn't traverse out of `config.ImgPath`.

But what if `reqURI` starts directly with `../`? After some experimentation, it's easy to see that in this case `../` will be retained directly, and then spliced with `config.ImgPath` to achieve directory traversal.

![](https://s2.loli.net/2024/04/26/PZO2y3GkV1zhal5.png)

So is it possible for `reqURI` to start with `../`? The framework used by [`webp_server_go`](https://github.com/webp-sh/webp_server_go) is [`fiber`](https://github.com/gofiber/fiber), while [`fiber`](https://) is based on [`fasthttp`](https://github.com/valyala/fasthttp). [`fasthttp`](https://github.com/valyala/fasthttp) does not report errors for malformed HTTP requests with URIs that do not begin with `/`, but treats them as legitimate URIs. This means that CVE-2021-46104 is not fully fixed, and we can traverse to the root directory to read the flag by constructing the following malformed HTTP message.

```http
GET ../../flag.png HTTP/1.1
```

Now there is only one last question, how to send this message to the backend [`webp_server_go`](https://github.com/webp-sh/webp_server_go)? Looking at the front-end's `/pics/<string:path>` routing, we see that the `accept` parameter passed to the `fetch_converted_image()` method takes the URL-decoded `Accept` request header, which is spliced directly into the HTTP message in the `fetch_converted_image()` method. As a result, we can implement HTTP request smuggling by inserting the URL-encoded `\r\n\r\n` to truncate an HTTP message into two consecutive HTTP messages, the latter of which is fully controllable, and the `fetch_converted_image()` method ends up returning the response body of the last message. The `Accept` request header can then be constructed as follows to accomplish our goal.

```http
Accept: image/webp%0d%0a%0d%0aGET ../../flag.png HTTP/1.1
```

So the final complete process is to upload a random image, access that image and grab the HTTP message, modify the `Accept` request header as described above, and send a request for a Flag.

A final note on why this vulnerability is weak. First of all, the malformed HTTP message must be sent directly to [`webp_server_go`](https://github.com/webp-sh/webp_server_go), which can't be exploited once a reverse proxy like Nginx has checked the URI in the middle, which is why this question combines it with HTTP request smuggling. Secondly, the vulnerability can only read image files, because [`webp_server_go`](https://github.com/webp-sh/webp_server_go) will feed the read file to VIPS for processing, and as long as the read file is not a legitimate image, VIPS will report an error, and the attacker can't get the content of the file, which is also the reason why the flag in this question is an image; finally, the attacker needs to have a priori knowledge of the path of the image, otherwise it's hard to read a valid image.

This vulnerability has been reported to the developer, and has been fixed in version 0.11.3.