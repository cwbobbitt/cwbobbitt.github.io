---
title: Progress Dialogs within Addigy Self Service
date: 2023-07-11 11:00:00 -0500
categories: [addigy, swiftdialog]
tags: [addigy, swiftdialog]
---

## Addigy Self Service

<a href="https://addigy.com/self-service/" target="_blank">Addigy's Self Service</a> is a great way to provide a collection of items to users including software and scripts. In most cases, Self Service's functionality is sufficient for basic installers and scripts that do not require interaction or acknowledgement from users by showing simple "installing" and "complete" progress indicators within the main Self Service window. Unfortunately, this can become a problem when items involve certain scenarios such as large installers or long/nested subprocesses that can take some time.

With the basic Self Service functionality, users often believe that no progress is being made if the current progress is still "installing" after several minutes when in reality, dependency files are being downloaded or a long process is running in the background. 

![image-title-here](/assets/images/addigy-progress-dialogs/self_service_example.png){:width="100%"} 

## Enter swiftDialog

_<a href="https://github.com/bartreardon/swiftDialog" target="_blank">swiftDialog</a> is an open source admin utility app for macOS 11+ written in SwiftUI that displays a popup dialog, displaying the content to your users that you want to display._

I recommend going over to the swiftDialog Wiki to read over what swiftDialog is designed to do and how you can get started with a few examples.

I'll start by showing the end result of what we're trying to acheive and will then break it down into sections of the required components.

![image-title-here](/assets/images/addigy-progress-dialogs/demo.gif){:width="100%"} 

## Smart Software Items

We can break Smart Software Items down into two scenarios:
1. Installers and files hosted within the Smart Software config
2. Installers and files being pulled down from Addigy's File Manager or the internet (curl, installomator, etc).

For scenario 1, we'll assume that any installers selected within a Smart Software item will first be downloaded into their respective ansible folder and will not need a download progress indicator since the file will already be downloaded prior to the installation script execution.

For scenario 2, we know that we'll need to pull down the installer from a location on the internet and should display a download progress indicator for users. In this case, we will use Addigy's lan-cache download option, which also provides a built-in mechanism to print download progress to stdout.

## Public Software Items

This workflow can also be used for Public Software Items hosted by Addigy and can often be beneficial if you'd like to just use the package Addigy has uploaded without needing to download and upload the package yourself. As you'll see later, the script functions exactly the same as it is pulling the public software item using the same Addigy lan-cache tool used by Custom Software items.

## Variables

For the main variables, we have the Software Name, Icon ID, and Icon MD5 that are used.

- **software_name**: Name of the item you are installing that will be displayed to the user
- **icon_id**: If you would like to set the Dialog icon to the same Self Service app icon that you have uploaded to your Smart Software item, provide the Addigy ID of the icon file. Otherwise, it will default to the Mac Manage icon. 340x340px preferred.
- **icon_md5**: Expected md5 value of icon in file manager. Leave blank if not using custom icon file.

{% highlight shell %}
software_name="Zoom (5.8.1)"
icon_id="19887369-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
icon_md5="bbc2f8704f3bebf34659fa8432fbac69"
{% endhighlight %}

Taking inspiration from other swiftDialog projects like #baseline and #setupyourmac, the remaining configuration for installer files is handled via the configJSON variable where you can specify multiple files/installers if needed. Each entry is composed of the following variables:

- **display_name**: Name of the item you are installing/running/copying that will be displayed to the user. 
- **file_name**: Name of file being downloaded (ie. zoom.pkg, script.sh, file.zip)
- **file_id**: ID of file within file manager
- **install_type**: PKG, DMG, SCRIPT, or COPY
- **md5**: Expected md5 value of item in file manager
- **arguments**: Optional arguments such as copy destination or script argument. (Copy destination required)

{% highlight shell %}
configJSON='
{
    "items": [
        {
            "display_name": "Zoom Installer",
            "file_name": "Zoom.pkg",
            "file_id": "c7f363e4-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "install_type": "PKG",
            "md5": "bbc2f8704f3bebf34659fa8432fbac69",
            "arguments": ""
        },
         {
            "display_name": "Image Files",
            "file_name": "images.zip",
            "file_id": "19887369-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "install_type": "COPY",
            "md5": "7fc2038271967cf9997a7b15ed0ea274",
            "arguments":"/Users/${loggedInUser}/Desktop/"
        },
        {
            "display_name": "Activate Software",
            "file_name": "activate.sh",
            "file_id": "026687cb-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "install_type": "SCRIPT",
            "md5": "aa6bcdbeb35db41d01f49c0945ab5fae",
            "arguments":"5F3Z-2ede-90dd4-ww9Z"
        },
        {
            "display_name": "Google Chrome",
            "file_name": "chrome.dmg",
            "file_id": "01556a13-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
            "install_type": "DMG",
            "md5": "d4293101785335d7b3d78c82544fe9af",
            "arguments":""
        }

    ]
}
'
{% endhighlight %}

## Retrieving the App and Icon ID

For both Smart Software and Public Software items, we can pretty easily retrieve the necessary ids. I'm sure there are multiple ways to acheive this, but I first started with retreiving these while looking at the Network of Chrome Developer Tools and then eventually moved to using Addigy's API.

### Method 1: Developer Tools

For those that don't want to go the API route, you can get the necessary ids for both the App and Icon ID by doing the following:
1. Open the Files menu within the Catalog page
2. Open Devloper Tools (Command + Option + J) 
3. Open the Network tab
4. Search for the package or icon in question within Addigy
5. Select the 'query' result in the network tab
6. Select the Preview tab
7. Expand the results to find the necessary id. If you have multiple results, make sure you're selecting the correct id that corresponds to the correct filename.

### Method 2: Addigy File Manager API

While the developer tool method above is fairly easy and can be done by anyone, it can become confusing if you have multiple files with the same name (for whatever reason) and is just a few more steps in general. Due to that, I've resorted to using Addigy's File Manager API to quickly provide the exact information that I need.

![image-title-here](/assets/images/addigy-progress-dialogs/api_demo.gif){:width="100%"} 

Prerequisites:
- API v2 key with View Files rights - For File Manager API
- API v1 key - For Public Software API
- jq (https://stedolan.github.io/jq/)

I've put together a fairly simple script that will present you with the option to search File Manager or the Public Software Catalog. 

For scenario 1 described earlier, use the File Manager option to search for your file ids and then use it to search for the image you'll be using for your icon to get your Icon ID. You'll also be provided with the MD5 hash and who within your organization uploaded the file.

For scenario 2, use the Public Software option to search for the app you'd like to use. If a corresponding app is found, you'll be presented with the Name, File ID, File Hash, Icon ID, Icon Hash, and Org ID. Keep in mind that Addigy hosted items will have an org id of b152eb8d-4b4b-11e5-8081-bd44aaafc9a6. Any other org ids are items hosted by other organizations.

{% highlight shell %}
fileManagerSearch(){

secret=""

echo "What file are you looking for?"
read fileName

files=$(curl -s -X POST "https://api.addigy.com/api/v2/oa/files/query" \
-H  "accept: application/json" \
-H  "x-api-key: ${secret}" \
-H  "Content-Type: application/json" \
-d "{\"search_term\":\"${fileName}\",\"per_page\":10,\"page\":1,\"sort_field\":\"created\",\"sort_direction\":\"desc\"}" )

echo "The following file(s) were found:"
echo $files | jq '.items[] | { File_Name: .filename, Date_Created: .created, User: .user_email, ID: .id, MD5: .md5_hash}'
prompt
}

publicSoftwareSearch(){

	client_id=''
	client_secret=''

	request=$(curl -s "https://api.addigy.com/api/catalog/public?client_id=${client_id}&client_secret=${client_secret}")

	echo "What software are you looking for?"
	read softwareName

	software=$(printf "%s" $request | jq . | jq -c '.[] | select( .name | contains("'${softwareName}'"))')
	printf "%s" $software | jq '. | {Name: .name, Org: .orgid , App_ID: .downloads[].id, App_MD5: .downloads[].md5_hash, Icon_ID: (.software_icon | .id), Icon_MD5: (.software_icon | .md5_hash)}'
	prompt
}

prompt(){

choices=('Search File Manager' 'Search Public Software' 'Quit')

PS3='What would you like to do?
'
select answer in "${choices[@]}"; do
	case "$answer" in
		"${choices[0]}")
			fileManagerSearch
			;;
		"${choices[1]}")
			publicSoftwareSearch
			;;
		"${choices[2]}")
			exit;
			;;
		*)
			echo "Invalid Choice. Please select 1, 2, or 3."
			;;
	esac
done
unset PS3
}

prompt
{% endhighlight %}

I redacted the IDs and emails from the File Manager Search section of the recording, but they'll be displayed as well if you use the provided API code.
![image-title-here](/assets/images/addigy-progress-dialogs/api_example.png){:width="50%"} 

## Putting It All Together

Once you've collected and compiled all the relevant information, we can put it all together within a Smart Software item. Make sure to name your custom software item the same as your software_name variable so it can all execute within the same directory. Additionally, don't forget that the versioning will add the version number to the end of the software name for it's directory. For example, a software item with the name "Zoom" and version 1.0 will need to be named "Zoom (1.0)" in your configuration below.

At this point, all you'll need to do is paste the code below (along with your specific variables) in the "installation command" section of your custom software item. 

Don't forget to upload your Self Service icon as well so it is displayed within MacManage itself!

{% highlight shell %}
# Chris Bobbitt - July 2023 - SwiftDialog Progress Dialogs for Addigy Self Service
# https://cwbobbitt.github.io/

############ VARIABLES ############
###################################

software_name="" # Name of the smart software or public software item. Most likely "Zoom (1.0)"
icon_id="" # FileManager ID for icon, png preferred, 340x340
icon_md5="" # 

### JSON Config - controls items being downloaded/installed - See Example below###
## display_name: name to be displayed as title when running
## file_name: name of file being downloaded (Zoom.pkg)
## file_id: id of file within file manager
## install_type: PKG, DMG, SCRIPT, or COPY
## md5: expected md5 value of item in file manager
## arguments: arguments such as copy destination (required) or script arguments

configJSON='
{
    "items": [
        {
            "display_name": "",
            "file_name": "",
            "file_id": "",
            "install_type": "",
            "md5": "",
            "arguments": ""
        }
    ]
}
'

############ /VARIABLES ###########
###################################

### Computed Variables ###
addigyPath="/Library/Addigy/ansible/packages/$software_name/"
addigyFileManager="https://file-manager-prod.addigy.com/file/"
dialogBinary="/usr/local/bin/dialog"
commandFile=$(mktemp /var/tmp/Dialog.XXXXXX)
errorCommandFile=$(mktemp /var/tmp/error-Dialog.XXXXXX)
logFile=$(mktemp "/var/tmp/${software_name}_log.XXXXXX")
icon="${addigyPath}Icon.png"

### METHODS ###

# Check if dialog/swiftDialog exists and install if not present.
# Dialog Check - Adam Codega - https://github.com/acodega/dialog-scripts/blob/main/dialogCheckFunction.sh

dialog_check() {
    
    # Get the URL of the latest PKG From the Dialog GitHub repo
    dialogURL=$(curl --silent --fail "https://api.github.com/repos/bartreardon/swiftDialog/releases/latest" | awk -F '"' "/browser_download_url/ && /pkg\"/ { print \$4; exit }")
    # Expected Team ID of the downloaded PKG
    expectedDialogTeamID="PWA5E9TQ59"

    # Check for Dialog and install if not found
    if [ ! -e "/Library/Application Support/Dialog/Dialog.app" ]; then
    echo "Dialog not found. Installing..."
    # Create temporary working directory
    workDirectory=$( /usr/bin/basename "$0" )
    tempDirectory=$( /usr/bin/mktemp -d "/private/tmp/$workDirectory.XXXXXX" )
    # Download the installer package
    /usr/bin/curl --location --silent "$dialogURL" -o "$tempDirectory/Dialog.pkg"
    # Verify the download
    teamID=$(/usr/sbin/spctl -a -vv -t install "$tempDirectory/Dialog.pkg" 2>&1 | awk '/origin=/ {print $NF }' | tr -d '()')
    # Install the package if Team ID validates
    if [ "$expectedDialogTeamID" = "$teamID" ] || [ "$expectedDialogTeamID" = "" ]; then
        /usr/sbin/installer -pkg "$tempDirectory/Dialog.pkg" -target /
    fi
    # Remove the temporary working directory when done
    /bin/rm -Rf "$tempDirectory"  
    else echo "Dialog found. Proceeding..."
    fi
}

# Loop through and download the item(s) within configJSON
function download_files() {
    items_length=$(get_json_value "${configJSON}" "items.length")

    for (( i=0; i<items_length; i++ )); do
        display_name=$(get_json_value "${configJSON}" "items[$i].display_name")
        file_name=$(get_json_value "${configJSON}" "items[$i].file_name")
        file_id=$(get_json_value "${configJSON}" "items[$i].file_id")
        md5=$(get_json_value "${configJSON}" "items[$i].md5")

        # Check if file_id is specified in configJSON item and download file if present. If not present, use local package file downloaded to ansible directory. If local file also doesn't exist, display error message.
        if [ ! -z "$file_id" ] 
        then
            echo "Downloading ${display_name}"
            echo "progress: 1" >> ${commandFile}
            echo "message: Downloading ${display_name}" >> ${commandFile}

            sleep 2 

            # Utilize the --print-progress flag with lan-cache to capture the download progress 
            /Library/Addigy/lan-cache --print-progress download "${addigyFileManager}${file_id}" "${addigyPath}${file_name}" 2>&1 | grep --line-buffered "ProgressPercent" | while IFS= read -r line; do
                tokens=(${line})
                progress=${tokens[1]}
                echo "progresstext: ${progress}%" >> ${commandFile} && echo "progress: ${progress}" >> ${commandFile}
            done

            # Validate that expected md5 matches the actual md5 of the download file. Quit if md5 doesn't match.
            actual_md5=$(md5 -q "${addigyPath}${file_name}")
            
            if [ "$actual_md5" != "${md5}" ]; then
                display_error "${display_name}" "The md5 hash didn't match the expected hash."
                break
            else
                echo "progress: complete" >> ${commandFile}
                echo "message: Download Complete" >> ${commandFile}
                echo "md5 good"
            fi
        elif [ -z "$file_id" ] && [ -f "${addigyPath}${file_name}" ];
        then
            continue;
        else
            display_error
        fi
    done
}

# Process the items previously downloaded. PKG, DMG, SCRIPT, or COPY.
function process_items() {
    items_length=$(get_json_value "${configJSON}" "items.length")
    
    for (( i=0; i<items_length; i++ )); do
        display_name=$(get_json_value "${configJSON}" "items[$i].display_name")
        file_name=$(get_json_value "${configJSON}" "items[$i].file_name")
        file_id=$(get_json_value "${configJSON}" "items[$i].file_id")
        install_type=$(get_json_value "${configJSON}" "items[$i].install_type")
        arguments=$(get_json_value "${configJSON}" "items[$i].arguments")
        md5=$(get_json_value "${configJSON}" "items[$i].md5")
        
            case "$install_type" in 
                "PKG")
                    process_pkg "${display_name}" "${file_name}"
                    ;;
                "DMG")
                    process_dmg "${display_name}" "${file_name}" "${arguments}"
                    ;;
                "SCRIPT")
                    process_script "${display_name}" "${file_name}" "${arguments}"
                    ;;
                "COPY")
                    process_copy "${display_name}" "${file_name}" "${arguments}"
                    ;;
            esac
    done

    echo "quit:" >> ${commandFile}
}

function process_pkg(){
    echo "Installing ${1}"
    echo "message: Installing" >> ${commandFile}
    echo "progresstext: ..." >> ${commandFile}

    sleep 2 

    # Capture % complete from installer output and then use for progress bar advancement in dialog
    /usr/sbin/installer -pkg "${addigyPath}${2}" -target / -verboseR 2>&1 | while read -r -n1 char; do
            [[ $char == % ]] && keep=1 ;
            [[ $char =~ [0-9] ]] && [[ $keep == 1 ]] && progress="$progress$char" ;
            [[ $char == . ]] && [[ $keep == 1 ]] && echo "progresstext: ${progress}%" >> ${commandFile} && echo "progress: ${progress}" >> ${commandFile} && progress="" && keep=0 ;
        done || true

    echo "message: Install Complete" >> ${commandFile}

    sleep 2
}

function process_dmg(){
    echo "Installing ${1}"
    echo "message: Installing" >> ${commandFile}
    echo "progresstext: ..." >> ${commandFile}

    sleep 2

    VOLUME=`hdiutil attach -nobrowse "${addigyPath}$2" | grep Volumes | sed 's/.*\/Volumes\//\/Volumes\//'`
    cd "$VOLUME"
    \cp -rf *.app /Applications
    INSTALLED=$?
    cd ..
    hdiutil detach "$VOLUME"

    echo "message: Install Complete" >> ${commandFile}

    sleep 2
}

function process_script(){
    echo "Running ${1}"
    echo "message: Running script" >> ${commandFile}
    echo "progresstext: ..." >> ${commandFile}

    sleep 2

    # Credit to BigMacAdmin (https://github.com/BigMacAdmin) within https://github.com/SecondSonConsulting/Baseline
    # https://superuser.com/questions/1066455/how-to-split-a-string-with-quotes-like-command-arguments-in-bash
    scriptArguments=${3}
    currentArgumentArray=()
    if [ -n "$scriptArguments" ]; then
        eval 'for argument in '$scriptArguments'; do currentArgumentArray+=$argument; done'
    fi
    
    sh "${addigyPath}${2}" ${currentArgumentArray[@]}

    echo "message: Script Complete" >> ${commandFile}

    sleep 2
}

function process_copy(){
    echo "Copying file ${2} to ${3}"
    echo "message: Copying" >> ${commandFile}
    echo "progresstext: ..." >> ${commandFile}

    sleep 2

    if [ -z "${3}" ]
    then
        display_error "${1}" "No copy destination was specified"
    else
        # Handle file extensions
        case "${2##*.}" in 
            zip)
                unzip "${addigyPath}${2}" -d ${3}
                ;;
            .gz)
                tar -zxvf "${addigyPath}${2}" -C ${3}
                ;;
            .tar)
                tar -xvf "${addigyPath}${2}" -C ${3}
                ;;
            *)
                cp "${addigyPath}${2}" ${3}
                ;;
        esac
    fi

    sleep 2
}

# Credit to Dan Snelson SetupYourMac (https://github.com/dan-snelson/Setup-Your-Mac/blob/main/Setup-Your-Mac-via-Dialog.bash)
function get_json_value() {
    JSON="$1" osascript -l 'JavaScript' \
        -e 'const env = $.NSProcessInfo.processInfo.environment.objectForKey("JSON").js' \
        -e "JSON.parse(env).$2"
}

function launch_dialog() {

    echo "Launching Dialog"

    title="${software_name}"
    message="Please wait..." 
    
    dialogCMD="$dialogBinary \
    --mini \
    --progress 100 \
    --title \"$title\" \
    --message \"$message\" \
    --icon \"$icon\" \
    --commandfile \"$commandFile\" "
    
    eval "${dialogCMD}" & sleep 0.3
}

function display_error() {
    echo "quit:" >> ${commandFile}
    echo "Launching Dialog"

    errorTitle="An Error Has Occurred"
    errorMessage="An issue has occurred while trying to process ${1}. ${2}. Please contact your IT Team." 
    errorIcon="SF=exclamationmark.triangle.fill"
    
    dialogErrorCMD="$dialogBinary \
    --mini \
    --title \"$errorTitle\" \
    --message \"$errorMessage\" \
    --icon \"$errorIcon\" \
    --commandfile \"$errorCommandFile\" "
    
    eval "${dialogErrorCMD}" & sleep 0.3

    finish_error
}

# Clean up command files and Addigy ansible directory.
function finish_success() {
    echo "Finishing"

    rm $logFile
    rm $commandFile
    rm $errorCommandFile
    rm -rf $addigyPath
}

# Clean up command files and Addigy ansible directory.
function finish_error() {
    echo "Finishing with error"

    rm $commandFile
    rm $errorCommandFile
    rm -rf $addigyPath
}

# MAIN COMMAND

# Download Self Service App icon to be used in Dialog with lan-cache. If not specified or md5 doesn't match, use standard MacManage icon. Can be replaced with any other file.
/Library/Addigy/lan-cache --print-progress --md5="${icon_md5}" download "${addigyFileManager}${icon_id}" "${addigyPath}Icon.png" 
[ $? -eq 0 ] && continue || icon="/Library/Addigy/macmanage/atom.icns"

dialog_check
launch_dialog
download_files
process_items
{% endhighlight %}

## Closing Thoughts and Future Plans

Thank you to the Addigy team and <a href="https://www.linkedin.com/in/ross-matsuda/" target="_blank">Ross Matsuda</a> for helping review and provide feedback on this!

My initial goal for this was solely for Addigy hosted items, but I do hope to add in support for curl and Installomator to further expand functionality. While this solution isn't perfect, I'm always open to feedback and recommendations for improvements and new use cases. 

If you have any questions or suggestions, feel free to shoot me a message over on the <a href="https://macadmins.slack.com/team/U011JUC8SLE" target="_blank">MacAdmins Slack</a>.
