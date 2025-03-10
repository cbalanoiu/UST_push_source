# UST push source
Obtain a csv list of accounts to use as source for 'push' stratergy of UST, by querrying the AD adudit log for events like: add to group, remove from group, enable and disable account

## Requirements:
- at least Python 3 installed
- PyYAML module installed (```pip install PyYAML```)
- access to run a PowerShell script on the AD machine
- resolve any permission needed to access the AD audit log for the running account

## Configuration
For the ease of this setup, the following setup assumes the scripts are going to reside inside User Sync Tool's folder, which runs on the AD machine.  

### extractADevents.ps1  
- modify how long in the past the script should look for events (below, the '-0.5' means half an hour ago)
```powershell
$Begin = (Get-Date).AddHours(-0.5)
```
- modify which properties should be extracted for the user identified in the event, in case the provided list is not sufficient:
```text
"mail", "Country", "sn", "GivenName", "UserPrincipalName", "MemberOf", "Enabled"
```
- modify which object properties should point to what variable. As a 'default' the script contain the ```username``` and ```domain``` values initialised as empty strings. For UST sync, this means Username=Email filed value in Admin Console. If these need to be different, make sure ```email``` and ```username``` point to the right variables holding the needed values. You will also notice the *customAttribute1-4*, which do not have proper mapping - use them in conjunction with any custom attribute you might need extracted (see previous bullet-point).  
- the ps script hardcodes the resulting csv events file to ```events.csv```; if needed, change the values in the last 2 lines of the script  

### prepare_push_list.py  
- if you modified the *events.csv* file in the ps script, make the same change for ```EVENTS_FILE_PATH```
- ```PUSH_FILE_PATH``` is initialised to ```push_list.csv``` as name of the output csv file of this script, which will be used as source for UST later on
- ```UST_FILE_PATH``` is targeting the ```user-sync-config.yml```; as mentioned, this script file is in same folder as UST, so no need to use absolute path
- ```LOGS_FOLDER``` is not initialised with a name, but since this run on a Windows machine, it could be modified to UST's usual logs folder, in this format: ```'C:\\path\\to_UST_folder\\logs\\'```; use the double backslashes!  

This Python script will filter the accounts appearing in the events, so that they are only the ones that need a change in Admin Console

### pushUMAPI.bat  
This is a sample file that contains the suite of scripts to be run in order to have full automation:  
- line 3: input the path where the UST folder containing the ps script is located
- line 6: document the full path to the ```extractADevents.ps1``` file
- line 10: if *Python* is installed and recognised as a global command, it does not need any change, otherwise full path to ```python.exe``` is required
- line 17: for version of UST >= 2.6.0, the provided command line and arguments (```user-sync.exe -t --strategy push --users file push_list.csv --process-groups```) runs a test-mode instance of the UST, targeting the ```push_list.csv``` file. Remove *-t* if changes are to be applied in Admin Console. Replace ```push_list.csv``` with the actual push list csv file name if you modified it in the ```prepare_push_list.py``` script.  

## What will happen
- the PS script will extract from AD's audit logs the events 4722, 4725, 4728, 4729 (enable/disable account, add/remove to/from group) for the past given time
- the PS script will produce an events csv file that will serve as input for the prepare_push_list Python script
- prepare_push_list.py will run and filter the accounts found in the event, so that only the ones that need to be pushed in Admin Console for a change are part of the resulting csv file. These accounts suffered a group change and that group is also mentioned as an User Sync Tool mapped LDAP group name. Disabling or enabling the account can also make it appear or disappear in/from one of same LDAP mapped groups and the script accounts for this scenario as well.
- with the resulting ```push_list.csv``` file, UST is run and the changes are forced in Admin Console: new accounts get added to mapped group(s), old accounts get added or removed from mapped groups, suspended accounts get removed from all mapped groups.  
 No account removal from Admin Console's Users menu happens, just group membership processing.
 
 # Known issues
 - the PowerShell script only works with events for Global Security groups, not Universal; those groups are also mapped in UST's ```user-sync-config.yml``` file
 - the script was tested for an AD forest with one domain tree


