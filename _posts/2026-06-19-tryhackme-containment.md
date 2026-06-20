---
layout: single
title: TryHackMe - ContAInment
categories: writeups tryhackme
toc: enabled
toc_sticky: enabled
---

This write up covers the room [ConAInment](https://tryhackme.com/room/containment), the final challenge of the AI Fundamentals module in the AI security learning path.

# Challenge
The scenario is as follows: One morning you arrive to work only to discover a workstation has been the target of a ransomware attack, the task being to investigate the incident by identifying the acces methods and actions of the attackers, affected data, and containing the threat.

For this task we are provided with a lab machine acting as the affected workstation, SSH credentials to interact with it via an AttackBox or VPN, and an "AI IR security assistant" hosted on the affected machine that is able to run tools to assist with the investigation and response. Interaction with the AI assistant is done through a web interface.

# Walkthrough
I began by SSH'ing into the target machine. Upon logging in I listed the user's home directory and got the following folder structure:
~~~
/home/o.deer
    ├── Desktop
    ├── Documents
    ├── Downloads
    ├── Mail
    ├── Music
    ├── Pictures
    ├── Public
    ├── Templates
    ├── Videos
    ├── alarms
    ├── qwen-output
    └── westtech_projects_encrypted.zip
~~~

Two objects of interest are a directory named ```qwen-output```, which I assume is related to the AI assistant, and ```westtech_projects_encrypted.zip```, where the flag of the room must be.

Exploring the directories I find the following information:

- ```Desktop``` contains the ransom note ```pwned.txt```.
- ```Downloads``` contains a file with the name ```invoice_payload.src```.
- ```Documents``` contains PCAP files, sorted in directories by date.
- ```Mail``` contains several pieces of mail.
- ```alarms``` contains ```soc_alarms```, which contains log files sorted into directories by date.
    - The directory ```2025-06-17``` contains the log file ```exfiltration_detected_1.log```

I proceeded to use the AI assistant. I instructed it to scan ```/home/o.deer/Mail``` with the ```phishing_email_detector``` tool. It found a phishing email with the subject ```INVOICE - URGENT REVIEW REQUIRED```. Checking the file ```2025-06-17_invoice_required_review.eml``` in the ```Mail``` directory confirms that this was a phishing email with the attachment ```invoice_payload.src```. Compiling this information and the contents of the other files of interest we can confirm the following facts:

- The incident took place on June 17th, 2025.
- The attacker gained gained access through a phishing email.
- The victim downloaded and opened the attached file.
- The malicious file downloaded a payload that allowed a reverse shell and exfiltrated data.

With this information I proceeded to check the PCAP dumps of the relevant date in ```Documents/pcap_dumps```. The files were unintelligible, except for ```session_4444_dump.pcap``` that seemed to contain fragments of a note left by the attacker. I asked the AI assistant to reassemble this file, saving the result in the ```qwen-output``` directory. The reassembled file contained the password for the ```westtech_projects_encrypted.zip``` file.

Unzipping the file reveals the recovered files, two of them being ```thm_flags.txt``` and ```thm_flags_guide.txt```. ```thm_flags_guide.txt``` contains the instructions for obtaining the flag: the file ```thm_flags.txt``` contains 500 possible flags encoded in base64, each one containing five comma separated two-digit number, the correct flag being the one with exactly 3 prime numbers.

To find the last flag the instructions encourage the use of the AI assistant, but I instead opted for writing a simple Python script:

~~~python
import base64
import re

# Load file with the encoded flags
file = open("thm_flags.txt")
primes = []

# Generate primes from 2 to 99
for i in range(2,100):
    is_prime = True
    for prime in primes:
        if i % prime == 0:
            is_prime = False
            break
    if is_prime and i not in primes:
        primes.append(i)

# Find the flag
for line in file:
    # Decode, separate each 2-digit number and parse
    string_bytes = base64.b64decode(line).decode("utf-8")
    numbers = re.findall(r"\d\d", string_bytes)

    primes_in_flag = 0
    
    # Check each number for primes
    for number in numbers:
        number_int = int(number)

        if number_int in primes:
            primes_in_flag += 1

    # The correct flag has exactly 3 primes
    if primes_in_flag == 3:
        print(numbers)
        break
~~~

The script first computes all prime numbers under 100. Then decodes each line and uses regex to extract each 2-digit number in the possible flag, and finally counts how many are prime, printing the flag with exactly 3 prime numbers.

# Summary
In summary, to obtain the flag:
- Instruct the AI assistant to identify the phishing email, taking note of the date it was delivered.
- Analize the PCAP files of the date of the incident and have the AI assistant reassemble the one with legible fragments.
- Analisis of the reassembled PCAP file reveals the necessary information to decrypt the data.
- Finally, with the use of AI or the Python script above, the flag can be found among the decrypted files.

# Incident Report
As an extra exercise I decided to write an incident report of this room.

## Executive Summary
The organization West Tech experienced a security incident on June 17th, 2025, during which an individual gained unauthorized access to sensitive data. Several internal project were encrypted and exfiltrated under a ransom, as well as employee sensitive data. The incident is now closed and a thorough investigation has been conducted.

## Timeline
On June 17th, 2025, an employee received an email from an external email. The email sender presented themselves as a service provider, threatening suspension of services if an file was not reviewed. The employee assumed the email was legitimate and opened the attached file.

Early the next morning internal monitoring systems flagged unusual network activity originating from the employee's workstation.

Later in the morning the employee notified the security team of the incident upon discovering a ransom note on their workstation.

## Investigation
The root cause was determined to be the email attachment. This file disguised it self as a terminal window attempting to open a PDF file, but in reality the file executed a script to download a malicious payload that enabled a reverse SSH shell, through which the attacker was able to access the workstation, encrypt data and exfiltrate it.

Analysis of logs revealed that the attacker was able to obtain the employee's sensitive data through an AI assistant hosted on the affected workstation through careful prompt engineering. However, it was also discovered that the attacker left the encryption key by accident on the logs, allowing the recovery of the affected data.

## Response and Remediation
The organization worked with public relaitonships to disclose the incident to stakeholders. Additionally, free identity protection services were offered to the affected employees.

IP addresses related to the exfiltration have been blocked.

## Recommendations
To prevent future recurrences, we are taking the following actions:
- Implement stronger access controls to sensitive data for AI assistants to prevent disclosure of information.
- Implement network security controls to detect and prevent suspicious network activity.
- Train employees in security awareness to reduce the effectiveness of phishing campaigns.

# Final Thoughts
This room showcased how AI can help in cybersecurity by taking care of mudane, repetitive tasks like searching for emails that look suspicious or analyzing logs for information, allowing for the security personel to focus on more important tasks. But also how it presents a considerable security risk when the proper guardrails are not implemented.

AI is just another tool that can do only as much damage as you allow it to. If AI is to be used in any capacity, care should then be put into implementing the proper security measures, like one would do any other program or human, in order to mitigate potential security risks.