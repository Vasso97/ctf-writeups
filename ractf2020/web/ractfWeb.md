## ractf All Web Challenges by sAINT_barber
---


### C0llide

Description:

`A target service is asking for two bits of information that have the same "custom hash", but can't be identical. Looks like we're going to have to generate a collision?`


By reading the description we understand that there will be some sort of collision between 2 or more hashes. After opening the page we are show the source code of the website:

```
const bodyParser = require("body-parser")
const express = require("express")
const fs = require("fs")
const customhash = require("./customhash")

const app = express()
app.use(bodyParser.json())

const port = 3000
const flag = "flag"
const secret_key = "Y0ure_g01nG_t0_h4v3_t0_go_1nto_h4rdc0r3_h4ck1ng_m0d3"

app.get('/', (req, res) => {
    console.log("[-] Source view")
    res.type("text")
    return fs.readFile("index.js", (err,data) => res.send(data.toString().replace(flag, "flag")))
})

app.post('/getflag', (req, res) => {
    console.log("[-] Getflag post")
    if (!req.body) {return res.send("400")}
    let one = req.body.one
    let two = req.body.two
    console.log(req.body)
    if (!one || !two) {
        return res.send("400")
    }
    if ((one.length !== two.length) || (one === two)) {
        return res.send("Strings are either too different or not different enough")
    }
    one = customhash.hash(secret_key + one)
    two = customhash.hash(secret_key + two)
    if (one == two) {
        console.log("[*] Flag get!")
        return res.send(flag)
    } else {
        return res.send(`${one} did not match ${two}!`)
    }
})

app.listen(port, () => console.log(`Listening on port ${port}`))
```

At the /getflag post route 2 parameters must be provided in the body of the request, both must have the same length and they cannot be the same. Otherwise we get a response of:

`Strings are either too different or not different enough`

I intercepted the request with burp, changed the request method to POST and added a `Content-Type: application/json` header so i can provide json in my request body and sent 2 strings, we get the response we expected.

![alt text][image1]

Sending two strings that match the above requierments we get both hashes of the supplied strings.

![alt text][image2]

But because the description metioned about the website using a custom hash,( + line 4), lets try and create a collision between both hashes. I first tried null bytes and arrays but nothing worked. Then i tried sending two empty objects:

![alt text][image3]

And we get the flag

flag: `ractf{Y0u_R_ab0uT_2_h4Ck_t1Me__4re_u_sur3?}`


[image1]: ./images/image1.png
[image2]: ./images/image2.png
[image3]: ./images/image3.png


### Quarantine - Hidden information
---
Description:

`We think there's a file they don't want people to see hidden somewhere! See if you can find it, it's gotta be on their webapp somewhere...`

From the description we check `/robots.txt` on the website and we notice one dissalowed entry:

`/admin-stash`

A simple get request to this directory reveals the flag

`flag: ractf{1m_n0t_4_r0b0T}`

### Quarantine
---
Description:

`See if you can get access to an account on the webapp.`

After reviewing the website a bit we notice and admin section that we can not access, a login page and i register page. The register page has not been implemented yet.

I tried a basic sql injection `' or 1=1;-- -` on the login page but we get this error:

![alt text][image4]

`Attempting to login as more than one user?`

Looks like our payload worked, but it triggered an error when it saw more that 1 row being return by the database, we can use `LIMIT` to limit the amount of rows that are returned, lets try:
`' or 1=1 limit 1;-- -`

![alt text][image5]


[image4]: ./images/image4.png
[image5]: ./images/image5.png

