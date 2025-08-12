---
title: CyberXbytes Web challenges
published: 2025-08-10
description: 'My walkthrough in web challenges on CyberXbytes'
image: 'https://media.licdn.com/dms/image/v2/D4E16AQHpUvBouzO2Mg/profile-displaybackgroundimage-shrink_350_1400/B4EZdP_GtFHcAY-/0/1749393633136?e=1757548800&v=beta&t=jQ9KXvzoDXi-PXC858ZBnDB-6DTamSKT_0ZHcUoM35A'
tags: [ctf, web]
category: 'CTF'
draft: false 
lang: 'en'
---

I was scrolling on X and saw a tweet about [CyberXbytes](https://cyberxbytes.com/) being published and hosting some challenges, so i decided to take a look.
***
# Silent Bypass
:::important
Bypass Me if U can (:
:::
The first challenge was medium level and had 6 solvers (i was the seventh)
![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-11%20204934.png)

I opened the challenge, and it was a login page...
![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20113232.png)

When i face login pages, i usually focus on three things:
- Default creds
- Injections
- Request manipulation

But before that, i start fuzzing, JavaScript analysis in order to see if there are any other endpoints on the site or any leaking creds that can help me. After some time, i found three endpoints while fuzzing:
- admin => probably has the flag
- login => our login page
- otp => maybe we need to enter otp after a successful login
![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20114332.png)

Looks great, but how do we log in? I tried to look for the things that i mentioned above, but none of them worked. When you enter wrong credentials, there is an error saying: **Invalid credentials**.

I tried common usernames, hoping the error gonna change cause in this case, you may know that the username indeed exists, so we can use it and brute force for the password. But unfortunately, the error didn't change; it was a robust logging system.

After some time, i opened the devtools to look for JS files. There were no files, but i found this.

<figure>
  <img src="/blog/cyberxbytes/silent-bypass/Screenshot 2025-08-08 115804.png" alt="credentials">
  <figcaption style="text-align: center;"> Classic </figcaption>
</figure>

After using the above creds, we can log in now. Noticed you get redirected to an `/otp` endpoint. The otp requires 4 digits, and the site was checking it on the front end. I tried to manipulate that and add more digits to see what happens.

![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20120139.png)

To do this, you need to intercept the request through Burp (or any proxy you use) and add more digits in the otp parameter to see if the backend accepts it or not.

![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20140959.png)

Noticed the request was refused by the backend too (**invalid otp**). Okay, noticed we got a flask session after the login. The server may not even be checking the OTB, and you can totally bypass it. I tried to go directly to `/admin` endpoint after the login, thinking i could bypass the OTB without the need to provide it, but i was wrong.
![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20141819.png)

You probably get a special token after a successful otp request that you can use to enter the admin panel. The good thing is that the otp was just 4 digits, which is easy to brute force. 
I made a wordlist with this script that i made to generate digits with the width as you need:
```python frame="terminal" title="python"
def generate_wordlist(start, end, width):
    
    return [str(i).zfill(width) for i in range(start, end + 1)]

def main():
    try:
 start = int(input("Enter start number (e.g., 0): "))
 end = int(input("Enter end number (e.g., 1000): "))
 width = int(input("Enter width (e.g., 4 for 0001): "))
        
        if start > end:
            print("Error: Start number must be less than or equal to end number.")
            return
        if width < 1:
            print("Error: Width must be positive.")
            return
            
 wordlist = generate_wordlist(start, end, width)
 filename = "wordlist.txt"
        with open(filename, 'w') as f:
            for item in wordlist:
 f.write(item + '\n')
        print(f"Wordlist saved to {filename}")
        print("First few entries:")
        print('\n'.join(wordlist[:5]))  # Show first 5 for preview
        
    except ValueError:
        print("Error: Please enter valid integers.")
    except Exception as e:
        print(f"An error occurred: {e}")

if __name__ == "__main__":
 main()
```
I intercept the otp request again, send it to the intruder, and start brute forcing using the wordlist i just made.

![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20142733.png)
I noticed that after 10 requests, you got a different response saying **Too many attempts**, indicating there is a rate limit in place, stopping us from brute forcing. In this situation, you need first to know how is that rate limit works? Is it triggered after 10 false requests (which could be a problem, harder to bypass), or does it trigger because we just send the requests too fast? OR it may be just a filter in place but with no power (i don't know how to name it xD), sometimes developers add a rate limit to prevent brute forcing, but it doesn't actually stop brute force attacks. You may see an error like **Too many attempts**, but if you ignore it, continue the brute force even with the error message appearing, and once you hit the right otp, you are in!.

To know that we need to test it first. I start by sending the requests with **.5 sec** delay in between. You can do that by:
1. Navigate to the **Resource Pool** tab in intruder
2. Start with 10 requests and **500** milliseconds

![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20150808.png)

Start the attack and see what happens.

![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-08%20151051.png)

Noticed we don't get the "too many" error anymore, we just get the invalid one. By waiting until the correct otp hits, you will get the admin token and successfully get the flag

<div class="image-container">
  <img src="/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-07%20215745.png" alt="Alt text for the left image">
  <img src="/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-07%20220037.png" alt="Alt text for the right image">
</div>
I also made a script that do the same thing:

```python frame="terminal" title="python"
import requests
import time
import sys

TARGET_URL = "http://185.185.82.29:10002/otp" 
SESSION_COOKIE = "eyJvdHBfYXR0ZW1wdHMiOjAsIm90cF9sb2NrZWQiOmZhbHNlLCJ1c2VybmFtZSI6ImFkbWluIn0.aJXaEQ.TOPjMCZNHekejRSMo5edG8yoVY4"  
WORDLIST_PATH = "wordlist.txt"     

session = requests.Session()
session.cookies.set("session", SESSION_COOKIE)

print(f"target: {TARGET_URL}")
print(f"wordlist: {WORDLIST_PATH}")

try:
    with open(WORDLIST_PATH, 'r') as wordlist_file:

 otps = [line.strip() for line in wordlist_file if line.strip()]
 total_otps = len(otps)
        print(f"Loaded {total_otps}")

        if total_otps == 0:
            print("Wordlist is empty.")
 sys.exit(1)

        for i, otp in enumerate(otps):
            try:
                
 payload = {"otp": otp}
 response = session.post(TARGET_URL, data=payload)
                print(f"OTP: {otp} | Status: {response.status_code}")
                if "CyberXbytes" in response.text:
                    print(f"[+] SUCCESS! Correct OTP found: {otp}")
                    print(f"[*] Full Response:\n{response.text}")
 sys.exit(0)
                if "Too many attempts" in response.text:
                    print("\n[!] Rate limit")
 sys.exit(1)

            except requests.exceptions.RequestException as e:
                print(f"failed for OTP {otp}: {e}")

 time.sleep(0.5)

except FileNotFoundError:
    print(f"\n wordlist not found")
 sys.exit(1)
except Exception as e:
    print(f"\nerror: {e}")
 sys.exit(1)
```
![image](/blog/cyberxbytes/silent-bypass/Screenshot%202025-08-07%20173719.png)
***

# Obfuscated Echo
![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-11%20205459.png)

:::important
In this challenge, your goal is to find a clever way to access the content of flag.txt using a specific exploit. The objective is to reveal the flag by leveraging template injection techniques. Can you figure out how to use the right command to get the flag? so can you cat flag.txt
:::

:::caution
You will be surprised how i solve at the end!!
:::

This one was medium. Let's see how to solve it
![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-08%20211001.png)


At the first of the challenge, you will face an empty page with a massage: `Hello noname............` Since the description says it's a **template injection attack** which will ease on us, we just need to find the injectable parameter that we will use to implement the attack. You can start fuzzing for parameters here, but i added the `name` parameter directly, i just had the feeling it's the right one.

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-08%20212152.png)

When it comes to SSTI, you have 3 phases:

- Detect
- Identify
- Exploit

## Detect
First, we need to see if there is indeed an SSTI vulnerability or not. To detect that we have a bunch of formats to test, the most common ones are (`{{ }}`, `<% %>`).
I started with the curly brackets format, since we know the site is running on Flask (a Python framework)
![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-08%20214821.png)

So the template engine likely will be one of the Python templating libraries.

I start with this simple payload `{{7*7}}` which is supposed to get rendered by the engine as **49** if the vulnerability really exists.

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-08%20215542.png)

Cool! We got our output, but we didn't finish yet.

## Identify

We need to know what exactly the type of template engine we are dealing with, probably it's **Jinja2** or **Django**. To make sure, we need to inject another payload, something like 
`{{7*'7'}}` which is related to **jinja2**.

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-09%20203642.png)

Suddenly, we got an error. This indicates the debugger was open. The important thing here is that we got the template engine, it's **jinja2**, and we got a weird *SyntaxError* says unexpected char `'&'`. Where does that character come from?? I didn't even inject `&` in my payload at all, maybe there is some sort of WAF that replaces my characters??!

There was nothing suspicious in my payload except the single quotes `'`. So i tried to inject only the single quotes and see what happens.

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-09%20204745.png)

I noticed indeed the single quotes and the double quotes too are being sanitized by a blind WAF that i couldn't really see in the code provided by the debugger.
Things are actually getting tricky since single and double quotes are sanitized, cause we need them in our payload. We need to figure something out

## Exploit

One of the repos that i get my payloads reference from is : [swisskyrepo](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Python/#jinja2-debug-statement)

To make my exploit, i usually try to find **classes** that can help me to read files in the system, but since the quotes are sanitized, the exploit may not work due to syntax errors problems. We need an alternative way to read files in the system. 

One way to do that is to get access to the `__builtins__` attribute, and to access that, we need a global function. jinja2 has a global function called `get_flashed_messages()` that we can use to access the `__globals__` then `__builtins__` which has functions like `open()` that can be used to read files.

The other thing we need to use is the `attr` filter, which lets you access an attribute of an object by its name.
For example:
- `user.name` is the same as `user|attr('name')`

But as you can see, there are quotes at the name part above `'name'` which are gonna be filtered, that's why we need to use `request.args` to reference these attributes as variables instead, to get rid of the quotes and access them via the request module.
For example:
- `{{get_flashed_messages|attr(request.args.p1)}}&p1=__globals__` => `get_flashed_messages.__globals__`

By these techniques combined, we can make a full exploit.

Start by reading the `/etc/passwd` file by using this payload:
`{{%20get_flashed_messages|attr(request.args.p1)|attr(request.args.p2)(request.args.p3)|attr(request.args.p2)(request.args.p4)(request.args.p5)|attr(request.args.p6)()%20}}&p1=__globals__&p2=__getitem__&p3=__builtins__&p4=open&p5=/etc/passwd&p6=read`

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-11%20202910.png)

To get the flag, change the `/etc/passwd` part with `flag.txt` (i spent some time here to get the right name)

And we successfully got the flag.

![image](/blog/cyberxbytes/obfuscated-echo/Screenshot%202025-08-10%20221121.png)

The funniest thing is, after all of that, i noticed there is a bug or a mislead at the challenge (i don't know if it's intended). You can get the flag by just typing `flag.txt` in the name parameter, even if it's followed by chunk characters

<figure>
  <img src="/blog/cyberxbytes/obfuscated-echo/Screenshot 2025-08-10 221504.png" alt="Flag">
  <figcaption style="text-align: center;"> Flag </figcaption>
</figure>

***

# HackTHEPASS?

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20204752.png)

:::important
Register an account on the application. Identify a flaw within the forgot password functionality and login as user "hack.CyberXbytes@gmail.com". After successful login, you will see a flag displayed.
:::

This one was Hard level, let's take a look.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20210119.png)

A simple login page looks like the one that Facebook has. There are no JS files, so i started fuzzing to see if there are any other endpoints that we can access.
I used Intruder with `common.txt` wordlist.

The fuzzer gives me three endpoints.

1. console
2. register
3. forget-password

The console one looks interesting, revealing that the debugger may be left open. i accessed this endpoint.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20210352.png)

Yup, the debugger is left open.

I started creating an account to login. After  a successful login you will see this.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20211254.png)

We need access to `hack.CyberXbytes@gmail.com` in order to get the flag. Let's navigate to the`/forget-password` endpoint, which we're gonna use to reset the victim's password.

I saw that the reset function works by providing your email, then you will receive a reset link with a token that you can use to reset your password.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20211925.png)

I added my email and received a link (Noticed you need to provide a real email since probably there is a real SMTP server running on the system)

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20212301.png)

I noticed this token is 32 characters; it looks like an MD5 hash. We can make sure of that by generating our MD5 hash and comparing it with the provided one. But there is the thing that exactly gets hashed? What does the server use the hash for, and then use it as a reset token?

That's actually kinda wild, it's maybe the email? Maybe the email with a timestamp? Maybe the email with a secret and timestamp, which will be a real issue for us. You will never know what really gets hashed, but don't forget we have the debugger left open. We need to make some sort of error to trigger the debugger and see the code.

I intercepted the reset request and tried to play with the parameter, i deleted the whole parameter.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20213844.png)

Cool, we triggered an error, and we got what we want. I copied the response into the browser and found this code.

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20214052.png)

The token was the MD5 hash of just the email, great. Now we need to MD5 the victim's email and use the hash value as a token to reset  his password and take control over the account

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20215217.png)

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20215152.png)

And we got the flag

![image](/blog/cyberxbytes/hackthepass/Screenshot%202025-08-11%20215326.png)