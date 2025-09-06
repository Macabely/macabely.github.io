---
title: HTB Nocturnal Machine (Easy)
published: 2025-08-24
description: 'My walkthrough on how i pwned the Nocturnal machine on HackTheBox'
image: 'images/nocturnal/Pasted image 20250724155806.png'
tags: [htb, web, linux, machines]
category: 'HTB'
draft: false 
lang: 'en'
---

Here I'm gonna walk through how I solved the **Nocturnal** machine. It's a Linux machine, an _easy_ one from HackTheBox.

***
# I Reconnaissance

First i started with a simple Nmap scan and found the following:

![Desktop View](images/nocturnal/Pasted%20image%2020250723154047.png)

As you can see there is an SSH server and web server running on port 80. By pasting the IP into the browser, you will get a domain called `nocturnal.htb`. Add the domain in the `/etc/hosts` file with the intended IP and you will get a vision on how the site looks:

![Desktop View](images/nocturnal/Pasted%20image%2020250723145604.png)

***
# II Analyze

It's a simple PHP application with a `login/register` function. Going to the register function at `/register.php` to create an account, then logging in at `/login.php` with the creds we just made.

![Desktop View](images/nocturnal/Pasted%20image%2020250723145857.png)

After a successful login, we can see a simple upload function. By uploading any file you will get this error:

![Desktop View](images/nocturnal/Pasted%20image%2020250723150240.png)

I actually tried to bypass  that with double extensions and other stuff but with no luck, it seems robust. I intercepted the request through burp, changed the extension to `.pdf` and uploaded the file successfully. Now we can see our file exist here:

![Desktop View](images/nocturnal/Pasted%20image%2020250723150657.png)

By pressing at the link, the file gets downloaded automatically. After that i went to burp and saw this request:

![Desktop View](images/nocturnal/Pasted%20image%2020250723150837.png)

You actually could see it on the browser too:

![Desktop View](images/nocturnal/Pasted%20image%2020250723151003.png)

We see some interesting stuff here:
1. `username` parameter: maybe we can change it to another username and get accessed to unauthorized files we shouldn't see. Leading to **IDOR** vulnerability.
2. `file` parameter: We may try path traversal here, trying to read system files like `/etc/passwd` leading to **LFI** vulnerability.

First i tested the `username` parameter.
I created a new account with username "**best**", uploaded some files. Then i sent the request above (from the "test" account) and changed the username from "test" (account 1) to "best" (account 2)

![Desktop View](images/nocturnal/Pasted%20image%2020250723152117.png)

The request was successful (Noticed, you need to add a valid extension in the `file` parameter. Otherwise you will get an error), revealing files from another user. We successfully got an IDOR vulnerability. Now it's time for fuzzing on that username parameter to see if there any valid names that we can see their files too.  
_I tried first to test the `file` parameter, but you need to add a valid extension from the extensions mentioned above, making it worthless to test_.

I used ffuf here to fuzz on usernames with my wordlist using this command:
`ffuf -w usernames.txt -u http://nocturnal.htb/view.php?username=FUZZ&file=.pdf -H "Cookie: PHPSESSID=<YOUR-SESSIOn>" -fs 2985`  
_While testing on the `username` parameter, i found that when you add an invalid name you got a response **length: 2985**. So we need to filter that out with the `-fs 2985` flag_.

![Desktop View](images/nocturnal/Pasted%20image%2020250723153359.png)

And we got some hits. `admin` and `tobias` has no files, but `amanda` dose.

![Desktop View](images/nocturnal/Pasted%20image%2020250723153635.png)

By downloading the file and read it using LibreOffice (or you can just use `cat` command), you will find a password.

![Desktop View](images/nocturnal/Pasted%20image%2020250723153825.png)

I thought i can use that password on the SSH server, but i was wrong. So it has to be a password that i can use to login at `amanda` account on the site. 
Going back to the login page, logging in with `amanda` creds.  
We successfully logged in and we have admin privileges too!

![Desktop View](images/nocturnal/Pasted%20image%2020250723154342.png)

By navigating to the admin panel, found the following:

![Desktop View](images/nocturnal/Pasted%20image%2020250723154554.png)

We got some PHP files that we can include and view it's source code. I Choose the admin page and i saw a `view` parameter showed up

![Desktop View](images/nocturnal/Pasted%20image%2020250723154954.png)

Tried to test that parameter against `LFI`, but no luck. I copied all the code, paste it in vscode cause it's easy to read and found some interesting things:

```php frame="terminal" title="PHP"
function sanitizeFilePath($filePath) {
    return basename($filePath); // Only gets the base name of the file
}
```

First of all the `sanitizeFilePath()` function. It's using the `basename()` function on the file path to extract only the filename from a given path (`../../test.php` becomes `test.php`). Preventing path traversal from happening (and i am asking why i can't get path traversal xD).  
The most importantly, is the backup process:

```php frame="terminal" title="PHP"
if (isset($_POST['backup']) && !empty($_POST['password'])) {
    $password = cleanEntry($_POST['password']);
    $backupFile = "backups/backup_" . date('Y-m-d') . ".zip";

    if ($password === false) {
        echo "<div class='error-message'>Error: Try another password.</div>";
    } else {
        $logFile = '/tmp/backup_' . uniqid() . '.log';
       
        $command = "zip -x './backups/*' -r -P " . $password . " " . $backupFile . " .  > " . $logFile . " 2>&1 &";
        
        $descriptor_spec = [
            0 => ["pipe", "r"], // stdin
            1 => ["file", $logFile, "w"], // stdout
            2 => ["file", $logFile, "w"], // stderr
        ];

        $process = proc_open($command, $descriptor_spec, $pipes);
        if (is_resource($process)) {
            proc_close($process);
        }

        sleep(2);

        $logContents = file_get_contents($logFile);
        if (strpos($logContents, 'zip error') === false) {
            echo "<div class='backup-success'>";
            echo "<p>Backup created successfully.</p>";
            echo "<a href='" . htmlspecialchars($backupFile) . "' class='download-button' download>Download Backup</a>";
            echo "<h3>Output:</h3><pre>" . htmlspecialchars($logContents) . "</pre>";
            echo "</div>";
        } else {
            echo "<div class='error-message'>Error creating the backup.</div>";
        }

        unlink($logFile);
    }
}
```

1. The backup creation is created through POST method. First it checks on if the `backup` parameter present and if the `password` parameter is not empty (so you create a backup with a password).
2. Then it checks/Sanitize that password provided by the user using `cleanEntry()` function (we will get into that later). If it's not Sanitized, you will get an error otherwise you are good to go.
3. It creates the backup file on `/backups` having that form: `backup_` following by the date(Y-m-d) and a zip extension (`backups/backup_2025-02-11.zip`)
4. A temporary log file gets created.
5. After that we have a zip command: 
	- `zip -x './backups/*'`: Excludes the backups directory from the archive.
	- `-r`: Recursively includes all files in the current directory.
	- `-P $password`: Sets the password for the ZIP file (gets it from the password parameter after Sanitization ).
6. Then the command gets executed using `proc_open()` function, waits 2 sec and checks the logs to see if there any error happened through the command execution process.
7. The log file gets deleted.

We actually have a potential command injection vulnerability here we can use on the **password** parameter, but we need to bypass the `cleanEntry()` function first:
```php frame="terminal" title="PHP"
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];

    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) {
            return false; // Malicious input detected
        }
    }

    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}
```
The `cleanEntry()` function using a blacklist to sanitize the password, that's not really a smart move since blacklisting is less secure than whitelisting. They should use `escapeshellarg()` function or something to prevent command injection from happening.  
After some time analyzing the other PHP pages, i found this line of code in `register.php` page:

```php frame="terminal" title="PHP"
<?php 
session_start();
$db = new SQLite3('../nocturnal_database/nocturnal_database.db');

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $username = $_POST['username'];
    $password = md5($_POST['password']);

    $stmt = $db->prepare("INSERT INTO users (username, password) VALUES (:username, :password)");
    $stmt->bindValue(':username', $username, SQLITE3_TEXT);
    $stmt->bindValue(':password', $password, SQLITE3_TEXT);

    if ($stmt->execute()) {
        $_SESSION['success'] = 'User registered successfully!';
        header('Location: login.php');
        exit();
    } else {
        $error = 'Failed to register user.';
    }
}
?>
```

The server using SQLite3 as a DBMS and there is this little tiny file `nocturnal_database.db` that contains creds that we may use to access the SSH server. Also passwords gets MD5 hash which maybe crackable.  
So the goal here to use the command injection vulnerability to read the `nocturnal_database.db` file, revealing the password hashes.
The thing is `nocturnal_database.db` is a sqlite3 database file, we need to use sqlite3 to dump it. We can't just add sqlite3, it will not work. We can use bash here.  
The bash command should look like this:  
`bash -c "sqlite3 ../nocturnal_database/nocturnal_database.db .dump"`  
But spaces are blacklisted, how we gonna bypass this?
***

# III Exploit

## Escape Characters

Using escape characters may help us here. Replacing the space `" "` with tab "`\t`" could work .
The final payload will be:
`bash\t-c\t"sqlite3\t../nocturnal_database/nocturnal_database.db\t.dump"`  
_adding CRLF (`\r\n`)at first to start in a newline on left will be smart move, But the above payload should work fine_.

**Note:** If you gonna inject the payload through burp, u need to URL encoded first.
![Desktop View](images/nocturnal/Pasted%20image%2020250723165437.png)

And we successfully got the hashes, time to crack it. Using [CrackStation](https://crackstation.net/) only `tobias` hash was crackable, which is fine.  
Tried to access the SSH server with the creds i just got, and got the first flag.


![image](images/nocturnal/Pasted%20image%2020250723165809.png)


***

# IV ROOT

It's time for privilege escalation. The first thing i do is to use this command `sudo -l` to see what commands i have access to. Noticed i have no access on sudo. 
So it's `linpease.sh` time ([PEASS-ng/linPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS))
I opened my local server, downloaded the tool from the victim machine and run it

![Desktop View](images/nocturnal/Pasted%20image%2020250723170151.png)

While analyzing the results from `linpease.sh`, i noticed port **8080** was opened. Maybe another webserver is running?

![Desktop View](images/nocturnal/Pasted%20image%2020250723170532.png)

By establishing a tunnel through SSH to that port using this command:  
`ssh -L 4444:localhost:8080 tobias@nocturnal.htb`.  
(Note: i used port 4444 cause i was running burp on 8080)  
Then navigate to the localhost port 4000 on the browser, we see a login page

![Desktop View](images/nocturnal/Pasted%20image%2020250723170933.png)

Tried to use `tobias` creds but it didn't work. So i went back to `linpease.sh` results. I didn't find much, most of the interesting files were forbidden.
So what now? I went back to the login page again and start playing with it a little bit.  
Start using response manipulation, looking for leaked endpoints with no luck.
I start fuzzing on the username with the same `tobias` password and found out it was **admin** that worked for me as a username.

![Desktop View](images/nocturnal/Pasted%20image%2020250723171834.png)

**ISPconfig:** is like a control panel for managing web hosting services on Linux servers.  
For this type of stuff, you should look for the service version and see if there is any vulns related to it.
But how we can know the version? By navigating to the help tab, you can see the version

![Desktop View](images/nocturnal/Pasted%20image%2020250723172255.png)

Searching for vulns related to the version, found it's vulnerable to **CVE-2023-46818**. A PHP code injection vulnerability in the `records` POST parameter on `/admin/language_edit.php` endpoint:  
==> [ISPConfig PHP Code Injection Vulnerability](https://seclists.org/fulldisclosure/2023/Dec/2).  
I also found a POC that i can use to get a reverse shell from this CVE:  
==> [CVE-2023-46818 Python3 Exploit for ISPConfigPHP Code Injection Vulnerability](https://github.com/ajdumanhug/CVE-2023-46818).  
I downloaded the POC, run it and we got the root flag.

Root Flag:
![image](images/nocturnal/Pasted%20image%2020250723173009.png)
