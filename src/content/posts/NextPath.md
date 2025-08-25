---
title: HackTheBox (NextPath)
published: 2025-08-11
description: 'How I solved the NextPath web challenge on HackTheBox'
image: '/blog/htb/c/Screenshot 2025-08-10 192020.png'
tags: [htb, web, ctf]
category: 'Web'
draft: false 
lang: 'en'
---

I solved the **NextPath** challenge on HackTheBox, it's a medium one from [Jorian](https://x.com/J0R1AN). Let's see how you can solve it.

The challenge has the source code. I usually start looking into the challenge first, then i review the code. I think this makes things easier.

First, you will see a page that really has nothing on, a dead page.

![image](/blog/htb/c/Screenshot%202025-08-10%20193719.png)

When i face something like this, i usually start fuzzing, but since we have the source code, no need for that. let's view the HTML source first. I started looking for endpoints in JS files or from HTML comments or something.

![image](/blog/htb/c/Screenshot%202025-08-10%20194421.png)

I noticed it's a Next.js app and i found an api endpoint `/api/team?id=`. navigating to this api endpoint, i realized it's just giving images of the team. 

![image](/blog/htb/c/Screenshot%202025-08-10%20195047.png)

I start playing with this endpoint a little bit.
The first thing i thought SQLi vulnerability, but before testing on SQLi, i started changing that id to other numbers, maybe we get something useful.  
When i change the id to numbers above 3, i start getting this error.

![image](/blog/htb/c/Screenshot%202025-08-10%20195017.png)

That's not a SQLi vulnerability, that's an LFI one, the server gets the parameter's value to read internal files in the system. Maybe it's time to read the source code now.
While analyzing the `team.js` file, i found this code:
```js frame=terminal title="JavaScript"
import path from 'path';
import fs from 'fs';

const ID_REGEX = /^[0-9]+$/m;

export default function handler({ query }, res) {
  if (!query.id) {
    res.status(400).end("Missing id parameter");
    return;
 }

  // Check format
  if (!ID_REGEX.test(query.id)) {
    console.error("Invalid format:", query.id);
    res.status(400).end("Invalid format");
    return;
 }
  // Prevent directory traversal
  if (query.id.includes("/") || query.id.includes("..")) {
    console.error("DIRECTORY TRAVERSAL DETECTED:", query.id);
    res.status(400).end("DIRECTORY TRAVERSAL DETECTED?!? This incident will be reported.");
    return;
 }

  try {
    const filepath = path.join("team", query.id + ".png");
    const content = fs.readFileSync(filepath.slice(0, 100));

    res.setHeader("Content-Type", "image/png");
    res.status(200).end(content);
 } catch (e) {
    console.error("Not Found", e.toString());
    res.status(404).end(e.toString());
 }
}
```
Code does the following:

- `handler()` function checks if the id parameter exists or not; if not, it will throw a 400 error
- If the id parameter exists, the function will check on its format, making sure it contains numbers only by using the regex above `ID_REGEX=/^[0-9]+$/m`.
- Then the function will filter any dots or slashes that exist in the parameter, preventing us from path traversal.
- The last thing it will do is to make your request on the `/team` path, and it will append `.png` extension to it, making only the images accessible.
- The filepath gets limited to 100 characters only


The function is immune; there is no way we can do anything malicious using that parameter. But the thing is, when you think about it, there is literally nothing useful on the app except that parameter, it's the only one that exist, so it has to be something related to it.

Since the core attack has to be related to that parameter, the first thing i thought of is **parameter pollution**. One way to do this is either to duplicate the parameter and add one in the body of the request and one in the URL, OR duplicate the parameter and add both in the URL.

The first one didn't work for me (gave me the `no such file` error), but the second one gave me something interesting.

![image](/blog/htb/c/Screenshot%202025-08-10%20203206.png)

That looks promising. It gave me an `Invalid format` error, not the `no such file` one. This means the server is indeed reading the next parameter, but we need to break the first one so the server can read the second one. One way to do this is using **escape characters**:

![image](https://cdn.educba.com/academy/wp-content/uploads/2020/01/Escape-Sequence-is-C.png)

One of these characters is **CRLF injection** that we could use to break the parameter syntax and make a new line `\n\r` so the server escapes the first parameter and reads the second one, which we can inject anything into it, not worrying about the filter.

Now, by adding a new line `\n` in the first parameter, we need to **URL-encode** it first `%0a`. We can see that the server is now reading the second parameter.

![image](/blog/htb/c/Screenshot%202025-08-10%20204659.png)

Great, the only thing bothering us now is the `.png` extension. Remember when we said that the filepath is 100 characters only?!  
We can get use of that to omit the `.png` extension by adding a path traversal sequence `../` that will make the filepath reach 100 characters without the png extension.

`../../`.....`../etc/passwd`

![image](/blog/htb/c/Screenshot%202025-08-10%20205437.png)

Cool, now we need to read the flag. Wait a second, where does the flag even exist? To know that you need to run a shell on the challenge container (or read the docker file), so you need to run the challenge locally.  
It's a docker container, and you will see a file called `build-docker.sh`, just run that file with sudo (of course, Docker needs to be installed)

After running the container image, you need to get its id.  
You can do that by using this command: `sudo docker ps`, grab the id then run this command:  
`sudo docker exec -it <container_id> sh`

Noticed that the flag exists in the root directory and also exists in other places. i noticed that the root directory is restrictive 
![image](/blog/htb/c/Screenshot%202025-08-10%20211004.png)

You need to figure out a path that will reach 100 characters correctly to use it in your payload
![image](/blog/htb/c/Screenshot%202025-08-10%20192321.png)

`../../../`...`proc/40/task/46/root/`...`proc/40/root/flag.txt`

<figure>
  <img src="/blog/htb/c/Screenshot%202025-08-10%20211349.png" alt="Flag">
  <figcaption style="text-align: center;"> Flag </figcaption>
</figure>