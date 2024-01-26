---
title: Password Expiration Notifications with Addigy
date: 2023-10-12 11:00:00 -0500
categories: [addigy, swiftdialog]
tags: [addigy, swiftdialog]
---

![Swift Dialog](/assets/images/addigy-password-expiration-notifications/swiftdialog.png "Title"){:width="50%"}

## Introduction
For fleets that currently don't have a mechanism to notify the user when their IDP password is expiring or when their local password becomes out of sync with their IdP password, it can be a headache to support and maintain. With that in mind, two scripts were created to determine when a user's IdP password was set to expire. 

1. Direct Azure Graph API Integration
2. Addigy's Identity Password Last Set Date Fact

Note: Most of this was build around Azure being the IDP, but if you're using another solution, you can easily go the route of using just the "Addigy's Identity Password Last Set Date Fact" config outlined later.

## Direct Azure Graph API Integration

### Requirements

 - Azure App Registration with access to the Microsoft Graph API is needed to generate a clientID and secret to make a request to [Azure](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
 - Addigy data-key-client or other secrets management mechanism for your clientId and secret being accessed within script. 
     - Helpful data-key-client write up by [Ross Matsuda](https://www.sudoade.com/secure-variable-storage-and-recall-using-addigys-data_key_client/)
     - _Please note that this isn't the intended use of data-key-client, so Addigy may change it at any time._
 - Addigy MacManage or other notification mechanism like SwiftDialog or Yo Notifications

### Script (Custom Fact)

Below is a simple script created to collect the password last set attribute for an Azure user and then calculate how many days until the password expires based on a specified number of days that a password is good for. This can easily be adjusted to work with other MDM solutions by implementing your own secrets management. For Addigy specifically, this script is used as a custom fact in order to run regularly and to be able to use daysLeft as a custom fsact.

```bash
#!/bin/sh
loggedInUser=$(/bin/echo 'show State:/Users/ConsoleUser' | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}')
secret="$(/Library/Addigy/data_key_client -fileIdentifier="<FILE ID>" | tr -d \")"
clientID="$(/Library/Addigy/data_key_client -fileIdentifier="<FILE ID>" | tr -d \")"
tenant="$(/Library/Addigy/data_key_client -fileIdentifier="<FILE ID>" | tr -d \")"
domain="<DOMAIN>" ## SET YOUR DOMAIN HERE
azureUser="${loggedInUser}@${domain}"
scope="https://graph.microsoft.com/.default"

get_json_value() {
    JSON="$1" osascript -l 'JavaScript' \
        -e 'const env = $.NSProcessInfo.processInfo.environment.objectForKey("JSON").js' \
        -e "JSON.parse(env).$2"
}

callAzure(){
    #Retrieve token for main request
    getToken=$(curl --silent -X POST -d "grant_type=client_credentials&client_id=${clientID}&client_secret=${secret}&scope=${scope}" https://login.microsoftonline.com/${tenant}/oauth2/v2.0/token)
    accessToken=$(get_json_value "$getToken" "access_token")

    #Get User's Password Last Set Time
    getUser=$(curl --silent -X GET -H "Authorization: Bearer ${accessToken}" 'https://graph.microsoft.com/v1.0/users/'"${azureUser}"'/?$select=lastPasswordChangeDateTime')
    passwordLastSet=$(get_json_value "$getUser" "lastPasswordChangeDateTime")
    passwordLastSetConverted=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$passwordLastSet" "+%s")

    #Set Max Password Age
    maxPasswordAge=<INSERT MAX PASSWORD AGE IN DAYS>
    now=$(date +%F)
    today=$(date -j -f "%Y-%m-%d" "$now" "+%s")

    #Calculate the days since the password was changed
    daysSince=$(( ($today - $passwordLastSetConverted) / (60 * 60 * 24) ))

    #Calculate how many days until password expires
    daysLeft=$(($maxPasswordAge - $daysSince))

    #Clear database and insert daysLeft and last time request to Azure was made
    sqlite3 "/var/tmp/pwd_last_set.db" 'DELETE FROM DATA'
    sqlite3 "/var/tmp/pwd_last_set.db" 'INSERT INTO DATA (ID, LASTCHECK, DAYSLEFT) VALUES (1,'$(date +%s)','$daysLeft')'
}

checkTime(){
    #Check when last request to Azure was made for last password set
    lastCheck=$(sqlite3 '/var/tmp/pwd_last_set.db' 'SELECT LASTCHECK FROM DATA WHERE ID = 1' ) 
    hourAgo=$(date -v-1H +%s)

    #Check if last request was within or over an hour ago. If less than hour, return previous collected daysLeft. If over an hour, callAzure. This can be modified, but was done to limit the number of requests being sent to Azure across an entire fleet.
    if [[ $lastCheck -lt $hourAgo ]]; then
        callAzure
    else
        #Return daysLeft to be collected by custom fact and/or for notification use
        echo $(sqlite3 '/var/tmp/pwd_last_set.db' 'SELECT DAYSLEFT FROM DATA WHERE ID = 1' ) 
    fi
}

#Check if database exists and create if needed. If db already exists, call checkTime. If db doesn't exist, create and callAzure.
if [ -f "/var/tmp/pwd_last_set.db" ]; then
    checkTime
else
    touch /var/tmp/pwd_last_set.db
    sqlite3 "/var/tmp/pwd_last_set.db" 'CREATE TABLE DATA (ID INT PRIMARY KEY, LASTCHECK BLOB, DAYSLEFT INT)' 
    callAzure
fi

```

### What the Script Does
- Creates local database to track how many days until password expires and last execution time of call to Azure. This is done to limit how often the API call is made to prevent potential rate limiting and reduce the overall number of calls across a fleet.
- If last execution time in local database hasn't run in the last hour:
    - Retrieves clientID and secret via Addigy's data-key-client
    - Makes request to Azure Graph API to retrieve provided user's password last set attribute
    - Set days until password expires and execution time in local database
- If last execution time is within last hour:
    - Output latest days until password expires value from local database

## Addigy's Identity Password Last Set Date Fact

If you're using the Azure Identity integration, Addigy now supports pulling in several user attributes from Azure AD, which includes the Password Set Date. With this data pulled in by the existing Identity integration, it eliminates needing to call Azure directly like above. 

It is worth noting that this data is pulled when end-users sign in each time via Identity, so it can potentially be inaccurate if a user has changed their password, but not completed an Identity login since then.

### Requirements
- MDM Payload to deploy Identity Password Last Set Date fact to user's device
- Addigy MacManage or other notification mechanism like SwiftDialog or Yo Notifications

### MDM Payload
One helpful feature Addigy [provides](https://support.addigy.com/hc/en-us/articles/4403542462099-Use-Device-Facts-as-Custom-MDM-Profile-Variables) is the ability to use facts within payloads. This allows us to deploy the Identity Password Last Set value within a payload to a device. While that is a great start, we need a way to retrieve that value once on the device. Below is and excerpt from the mdm profile that was needed to deploy the "identity_password_last_set_date" fact to idpLastSet.plist within `/Library/Managed Preferences/`

{% highlight liquid %}
{% raw %}
<key>PayloadContent</key>
<array>
  <dict>
    <key>PayloadType</key>
        <string>idpLastSet</string>
    <key>Set Date</key>
        <string>{{.Fact "identity_password_last_set_date"}}</string>
  </dict>
</array>
{% endraw %}
{% endhighlight %}


### Identity Fact Script

 ```bash
loggedInUser=$(/bin/echo 'show State:/Users/ConsoleUser' | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}')
uid=$(id -u "$loggedInUser")
passwordLastSet=$(/usr/libexec/PlistBuddy -c "print 'Set Date'" /Library/Managed\ Preferences/idpLastSet.plist)
passwordLastSetConverted=$(date -j -f "%Y-%m-%d %H:%M:%S %z" $passwordLastSet +%s 2&>1)
maxPasswordAge=<INSERT MAX PASSWORD AGE IN DAYS>
now=$(date +%F)
today=$(date -j -f "%Y-%m-%d" "$now" "+%s")
daysSince=$(( ($today - $passwordLastSetConverted) / (60 * 60 * 24) ))
daysLeft=$(($maxPasswordAge - $daysSince))
echo $dayLeft
```

### What the Script Does
This script is a lot simpler than the Azure integration and simply retrieves the password last set date form the plist previously created and returns the number of days left until the password needs to be changed. 


## Notifications

All of these notifications scripts are designed to be used within a Addigy Maintenance item, but they can also be a run at your own cadence or preference. Both the Azure and Identity methods have their own way of retrieving localLastSet and daysLeft to be used in notifications, so utilize whatever method you'll be using within your full notification script (MacManage, SwiftDialog, Yo, etc).

#### Azure Notification Data

While the main script is used to store the number of days until the password expires as a Custom Fact within Addigy in order to display a notification with a Maintenance script, this script can be used anywhere to collect these metrics and then display a notification to a user.

```bash
loggedInUser=$(/bin/echo 'show State:/Users/ConsoleUser' | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}')
uid=$(id -u "$loggedInUser")
daysLeft=$(/usr/bin/sqlite3 /var/tmp/pwd_last_set.db "SELECT DAYSLEFT FROM DATA")
passwordLastSetConverted=$(/usr/bin/sqlite3 /var/tmp/pwd_last_set.db "SELECT LASTSET FROM DATA")
localLastSet=$(dscl . read /Users/"$loggedInUser" | grep -A1 passwordLastSetTime | grep real | awk -F'real>|</real' '{print $2}' | cut -d "." -f1)
```

#### Identity Fact Notification Data

This is almost identitcal to the custom fact script, but also collects the last time the local password was changed for additional logic.

 ```bash
loggedInUser=$(/bin/echo 'show State:/Users/ConsoleUser' | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}')
uid=$(id -u "$loggedInUser")
passwordLastSet=$(/usr/libexec/PlistBuddy -c "print 'Set Date'" /Library/Managed\ Preferences/idpLastSet.plist)
passwordLastSetConverted=$(date -j -f "%Y-%m-%d %H:%M:%S %z" $passwordLastSet +%s 2&>1)
localLastSet=$(dscl . read /Users/"$loggedInUser" | grep -A1 passwordLastSetTime | grep real | awk -F'real>|</real' '{print $2}' | cut -d "." -f1)
maxPasswordAge=<INSERT MAX PASSWORD AGE IN DAYS>
now=$(date +%F)
today=$(date -j -f "%Y-%m-%d" "$now" "+%s")
daysSince=$(( ($today - $passwordLastSetConverted) / (60 * 60 * 24) ))
daysLeft=$(($maxPasswordAge - $daysSince))
```

###  Mac Manage Notification

```bash
title="Password Expiration Alert"
description="Your password will expire in $daysLeft day(s). Please change your password as soon as possible."
acceptText="Change Password"
closeText="Close"
forefront="True"

function prompt(){
  if /Library/Addigy/macmanage/MacManage.app/Contents/MacOS/MacManage action=notify title="${title}" description="${1}"  acceptLabel="${2}" forefront="$forefront"; then
    echo "${2} clicked"
    case "${2}" in
      "Change Password")
        open "IDP PASSWORD CHANGE LINK" # Insert your IDP password change link. Commonly myaccount.microsoft.com for Azure
        ;;
      "Log out")
        /bin/launchctl asuser "$uid" /usr/bin/    osascript <<EOF
        try
          ignoring application responses
            tell application "/System/Library/CoreServices/loginwindow.app" to «event aevtrlgo»
          end ignoring
        end try
EOF
        ;;
    esac
    exit 0
  else
    echo "Close clicked"
    exit 0
  fi
}

if [[ "$daysLeft" -le "$maxPasswordAge" && "$daysLeft" -gt 0 ]]
then
    prompt "Your password will expire in $daysLeft day(s). Please change your password as soon as possible." "Change Password"
elif [[ "$daysLeft" -le 0 ]]
then
    prompt "Your password has expired. Please change it as soon as possible." "Change Password"
elif [[ "$passwordLastSetConverted" -gt  "$localLastSet" ]]
then
    prompt "Your local password is out of sync with your network password. Please log out and perform a full sign in into your Mac with your email to complete the full sync process." "Log out"
fi


```
![Mac Manage](/assets/images/addigy-password-expiration-notifications/macmanage.png){:width="75%"} 

###  Swift Dialog Notification

```bash
if [[ "$daysLeft" -le "$maxPasswordAge" && "$daysLeft" -gt 0 ]]
then
   launchctl asuser "$uid" /usr/local/bin/dialog --notification --title "Password Expiration Alert" --message "Your password will expire in $daysLeft days."
elif [[ "$daysLeft" -le 0 ]]
then
    launchctl asuser "$uid" /usr/local/bin/dialog --notification --title "Password Expiration Alert" --message "Your password has expired. Please change it as soon as possbile."
elif [[ "$passwordLastSetConverted" -gt  "$localLastSet" ]]
then
    launchctl asuser "$uid" /usr/local/bin/dialog --notification --title "Password Expiration Alert" --message "Your local password is out of sync with your network password. Please log out and back into your Mac with your email to complete the full sync process."
fi
```

![Swift Dialog](/assets/images/addigy-password-expiration-notifications/swiftdialog.png){:width="75%"} 

If you have any questions or suggestions, feel free to shoot me a message over on the <a href="https://macadmins.slack.com/team/U011JUC8SLE" target="_blank">MacAdmins Slack</a>.
