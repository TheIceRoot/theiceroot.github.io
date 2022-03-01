# Active Directoy Password Cracking

### Introduction
In this write-up, I will go over how to extract user credentials from the NTDS.dit file and crack the user's passwords. This will provide the analyst with credentials for all Active Directory (AD) users. Although you are reading this and probably asking, "Wouldn't this write-up help attackers?". The answer is yes, but acknowledging that this attack exists and how it works will allow you to implement preventative measures and protect your AD data.

### What is the NTDS.dit file?
The NTDS.dit file is a file created and used by AD for information including user credentials, groups, and group memberships. This file is continuously being used to record data therefore, copying or viewing the contents of this file will present the user with an error. Since this file contains a treasure trove of information, implementing monitoring and prevention policies will prevent attacks such as Mimikatz pass-the-hash attacks or pass-the-ticket. Oh, did I mention that this file is stored on the AD Domain Controller (DC)? (%SYSTEMROOT%\Windows\NTDS\ntds.dit). The domain controller should already be one of the most protected pieces of equipment in the organization, but it isn't, here is the hint to implement something.

### Analysis
#### Method 1
***Note: This analysis is done on an image of a domain controller and is not performed on a live system***

<img src="https://github.com/TheIceRoot/theiceroot.github.io/blob/main/images/ResearchImages/ADPasswordCracking/Figure%201.png" alt="Figure 1" width="auto" height="auto"><br>

The analyst loaded the DC image into Autopsy and was able to extract the SYSTEM hive (%SYSTEMROOT%\Windows\System32\Config\) by right-clicking the file and choosing the option "Extract File(s)".

<img src="https://github.com/TheIceRoot/theiceroot.github.io/blob/main/images/ResearchImages/ADPasswordCracking/Figure%202.png" alt="Figure 2" width="auto" height="auto"><br>

Similarly, the analyst extracted the NTDS.dit file (%SYSTEMROOT%\Windows\NTDS\) from the DC image in Autopsy.

```
git clone https://github/SecureAuthCorp/impacket.git

cd impacket/
```

The analyst installed "impacket" from a git repository which was the key tool in scraping the NTDS.dit file for NTLM ("Windows" New Technology LAN Manager) passwords. The analyst then changed into the impacket directory to run the python scripts that were imported.

<img src="https://github.com/TheIceRoot/theiceroot.github.io/blob/main/images/ResearchImages/ADPasswordCracking/Figure%203.png" alt="Figure 3" width="auto" height="auto"><br>

Using "secretdump.py", the analyst was able to use the NTDS.dit file and the private key from the SYSTEM hive to scrape AD credentials (as seen in Figure 3).

```
cat hashes.ntds | cut -d : -f 4 |sort|uniq > JustTheHashes.txt
```

Before decoding the NTLM password, the analyst removed everything except the NTLM hash value and only recorded unique hashes. The command seen above should do the trick.

<img src="https://github.com/TheIceRoot/theiceroot.github.io/blob/main/images/ResearchImages/ADPasswordCracking/Figure%204.png" alt="Figure 4" width="auto" height="auto"><br>

The analyst was finally left will all of the unique hashes of AD users. Now... time to get crackin'.

<img src="https://github.com/TheIceRoot/theiceroot.github.io/blob/main/images/ResearchImages/ADPasswordCracking/Figure%205.png" alt="Figure 5" width="auto" height="auto"><br>

Thankfully NTLM hashes aren't too hard to crack, especially when you can use an online cracker such as "CrackStation" (https://crackstation.net/). Finally, the analyst was left with the passwords for most of the AD users.

#### Method 2
There are more than two methods of cracking AD passwords using the NTDS.dit file, but for the sake of this write up, I will only go over two methods. (Will update in the future!)

```
PS: Install-Module -Name DSInternals -Force
```

To utilize PowerShell to scrape for AD user credentials, the analyst installed the DSInternals zip file (https://www.dsinternals.com/en/downloads/) and ran the command seen above to add the DSInternals DLLs to their machine. The reason for the '-Force' flag is because Windows does not like to add random DLLs to the machine. **Note**: PowerShell must be opened using administrative privileges for this to work.

```
PS: Get-BootKey -SystemHivePath [Path]
```

The analyst then used the "Get-BootKey" command in PowerShell to pull the private key from the SYSTEM hive. [Path] will be replaced with the path to the SYSTEM hive (which was copied in previous steps in Autopsy). 

```
PS: $key = Get-BootKey -SystemHivePath [Path]
```

Once the private key was successfully pulled from the SYSTEM hive, the analyst created a variable in PowerShell to call the private key. **Note**: In a perfect situation, the private key will pull from the SYSTEM hive file. If the SYSTEM hive is corrupted or "dirty", use esentutl to clean the database.

```
PS: Get-ADDBAccount -All DBPath [Path] -BootKey $key
```

Using the "Get-ADDBAccount" command, the analyst was able to pull all AD credentials from the NTDS.dit file. [Path] will be replaced with the path to the NTDS.dit file. Once the credentials have been listed, the analyst used "CrackStation", an online NTLM password cracking tool, to crack the AD user passwords.

### Conclusion
In conclusion, the analyst was able to successfully scrap Active Directory users' passwords from the NTDS.dit file utilizing multiple options. If the analyst had known the file paths to the SYSTEM hive and the NTDS.dit file, the extraction, and cracking of the passwords could have been performed offline, and thus, there would have no detection. This is very bad news as a blue teamer. As a red teamer/threat actor, I just got the golden ticket to laterally move around the system and access any AD account I want including domain admins. If I have access to the DC, I already won, but now I won bragging rights. Finally, it is worth noting to secure your environment and ensure there are certain security measures in place so that this does not happen. This can be as simple as allowing the least amount of access to the DC as possible and restricting access where needed.

#### Sources
1. https://www.ultimatewindowssecurity.com/blog/default.aspx?d=10/2017#:~:text=dit%20File%3F-,The%20Ntds.,all%20users%20in%20the%20domain.&text=The%20extraction%20and%20cracking%20of,so%20they%20will%20be%20undetectable. 

2. https://www.dsinternals.com/en/dumping-ntds-dit-files-using-powershell/ 

3. https://securitytutorials.co.uk/password-audit-extracting-hashes-from-ntds-dit/ 

4. https://crackstation.net/ 

5. https://github.com/SecureAuthCorp/impacket
