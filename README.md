# Overview
Users commonly use passwords that are predictable and easy to remember (i.e. Winter2017!, qwerty1!, password123). IT administrators constantly wage the cultural battle between convenience and security. If the password requirements are too complex, Post-it notes magically begin appearing under keyboards, taped to the monitor, or hidden in a drawer. 

[HaveIBeenPwned (HIBP)](https://haveibeenpwned.com/) is a website developed to aggregate recent data breaches and inform users if their credentials have been compromised. Although the site does not display email addresses and passwords together due to security concerns, the site also allows users to [check if their password has been compromised](https://haveibeenpwned.com/Passwords). The password hash list is available to download for integration into other systems. This is the premise of the following work. 

While it's possible to integrate [Active Directory (AD) password changes with HIBP](https://github.com/braimee/bpatty/blob/master/blue_team/pwnedpasswords.md), there's still a need for administrators to verify their current passwords are not present on a wordlist or commonly compromised by others. Service accounts are commonly set to never expire the password, so it's critical these are not easily compromised. This process will allow IT administrators a way to compare all accounts against 551 million previously compromised passwords. 

# Compare AD User Hash to HIBP
To perform this comparison, we will need to perform the following activities: 

1. Download the recent NTLM Pwned password list from HIBP
2. Create a copy of NTDS.dit and the HKLM\SECURITY registry hive
3. Extract the user hash from NTDS.dit and export it to a txt file
4. Compare the user hash to the hash list provided by HIBP

## Download the HIBP Pwned password list
[Download](https://haveibeenpwned.com/Passwords) the recent NTLM Pwned password list from HIBP. It's recommended to use your favorite torrent client instead of the direct download. Remember to seed if possible!
   * **NOTE:** There is a file ordered by prevelence and one ordered by hash. I haven't seen a significant speed difference between the two:
     ![alt text](https://github.com/mysecus/HaveIBeenPwned/blob/master/pics/Match%20Hash%20Speed%20Test.png "Hash Comparison Speed Test")

Once the file has been downloaded, transfer the extracted txt file to the domain controller. Due to [security concerns](https://blog.stealthbits.com/complete-domain-compromise-with-golden-tickets/), it's recommended to contain this process to the DC. This will be addressed later.


## Create a copy of ntds.dit and the HKLM\SECURITY registry hive
If you've ever tried navigating to %SystemRoot%\NTDS\ and copying the NTDS.dit file, you've probably experienced that it's in-use and unable to be copied. Instead of shutting down the domain controller, pulling the hard drive, and copying the file, we can use a built in utility to make a copy of it. 

1. Open PowerShell as an administrator on the domain controller and open the NT Directory Service Utility: 
    ```
    ntdsutil
    ```
2. Activate a new NT Directory instance: 
    ```
    activate instance ntds
    ```
3. Open NTDSUtil's Install From Media (IFM) tool. This will is used to create a copy of the AD Forest and System Registry: 
    ```
    ifm
    ```
4. Create a full dump of the AD Forest and Registry Keys to your desired location: 
    ```
    create full <DESIRED FILE PATH>
    ```
5. Once the files have been copied, quit IFM. 
    ```
    quit
    ```
6. Quit UTDSUtil
    ```
    quit
    ```

### Example process:

![alt text](https://github.com/mysecus/HaveIBeenPwned/blob/master/pics/NTDS.png "Create copy of AD Forest and Registry")

## Extract the user hash from NTDS.dit and export it to a txt file

### ![#f03c15](https://placehold.it/15/f03c15/000000?text=+) **Security Notice:** 

The following steps will expose the hash for the Kerberos account. If captured, attackers may then [obtain complete control into the network](https://blog.stealthbits.com/complete-domain-compromise-with-golden-tickets/), impersonate any user, and remain virtually undetectable. This is why it was recommended that the data does not leave the domain controller. All files generated during this process should be securely deleted once complete. 

### Prerequisite: 
Before extracting the hashes from ntds.dit, the [DSInternals](https://github.com/MichaelGrafnetter/DSInternals) module must be installed. 

1. To test if this module is already installed (it likely isn't), the following command will error out if DSInternals is missing:
    ```
    Get-Command Get-ADDBAccount
    ```
2. Install the DSInternals Module
    ```
    Install-Module DSInternals
    ```
3. You will be prompted to install from an untrusted repository. Continue installing the module to proceed. 

### Extracting the NTLM Hash
1. Using the files previously exported, set HKLM\SYSTEM as the boot key for ntds.dit
    ```
    $key = Get-BootKey -SystemHivePath '<DESIRED FILE PATH\registry\SYSTEM>'
    ```
2. Extract the NTLM hash for all users to a txt file.
    ```
    Get-ADDBAccount -All -DBPath '<DESIRED FILE PATH\Active Directory\ntds.dit>' -BootKey $key | Format-Custom -View HashcatNT | Out-File '<DESIRED FILE PATH>\UserHash.txt' -Encoding ASCII
    ```
    
### Example process:

![alt text](https://github.com/mysecus/HaveIBeenPwned/blob/master/pics/Extract%20Hash.png "Extract NTLM Hash")

## Compare AD Hash to HIBP Hash file
At this point, all local hash extraction is complete and ready for the comparison to the HIBP Pwned hash list. In a small test environment, this process took approximately 30 minutes to complete. I have not had the opportunity to run this in a prod environment yet. 

1. [Download the Match-ADHashes function](https://github.com/DGG-IT/Match-ADHashes/blob/master/Match-ADHashes.ps1)
2. Import the function by dot sourcing the file path
    ```
    . '<FILE PATH>\Match-ADHashes.ps1'
    ```
3. Begin the hash comparison
    ```
    Match-ADHashes -ADNTHashes ‘<PATH TO USER HASHES>’ -HashDictonary ‘<PATH TO HIBP TXT FILE>’
    
### Example process:

![alt text](https://github.com/mysecus/HaveIBeenPwned/blob/master/pics/Match%20Hash.png "Match NTLM Hash")

## Output

For an exciting project, the output is uneventful. There is no fancy HTML report, email notifications, or phone call from [Troy Hunt](https://haveibeenpwned.com/About). Instead, you're greeted with the username of the user with a compromised password, the frequency of compromises using this password, and the hash compromised. 

In the following example, my account was using the password ```qwerty1!```. This is enough to meet the complexity requirements for most organizations, but it's an insufficently poor password that's likely on every wordlist. 

![alt text](https://github.com/mysecus/HaveIBeenPwned/blob/master/pics/Final%20Output.png "Final Output")
