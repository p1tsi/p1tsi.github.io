---
title: MacOS LPE via NordVPN installation
date: 2024-09-30 18:25:00 +0200
categories: [macOS, bug bounty]
tags: [post]
---

## Introduction

Distributing macOS applications outside of the App Store is a common practice among software vendors. Nonetheless, it could be quite dangerous letting users download installation packages directly from the vendors' website and install them manually, especially with .pkg installers.

The vulnerability described in this blogpost is about a flaw I discovered in NordVPN installation package (version < 8.17), in particular inside preinstall and postinstall scripts. You can grab the latest version from [https://downloads.nordcdn.com/apps/macos/generic/NordVPN-OpenVPN/latest/NordVPN.pkg](https://downloads.nordcdn.com/apps/macos/generic/NordVPN-OpenVPN/latest/NordVPN.pkg).

`preinstall` and `postinstall` scripts are used to prepare the environment before and after the application's installation, for example, removing previous installation's files or moving privileged helper scripts inside the proper directory. Oh, yes... I was forgetting about one thing: they run on the system with root privileges. For this reason, when coming across a .pkg file, it is always a good idea to have a look inside it and check which commands are executed inside preinstall and postinstall scripts.

For this scope there exists an application called [Suspicious Package](https://mothersruin.com/software/SuspiciousPackage/). Or if you prefer to do things manually, at the end of this article there is a bash script that helps you to extract .pkg files.

## Root Cause Analysis

The NordVPN application bundle seems to be quite standard. The only thing to highlight is that it contains a privileged helper binary `com.nordvpn.macos.helper` which during installation will be placed inside the root-owned `/Library/PrivilegedHelperTools` directory. It obviously will run with root privileges.

These privileged helpers are launched loading the corrispective plist file inside `/Library/LaunchDaemons` or `/Library/LaunchAgents`. In NordVPN case, this plist file is created during the execution of postinstall script.

Finding a way to replace the real helper with a custom binary could let to execute arbitrary code with root privileges.


The postinstall script defines multiple functions and calls these 3 ones:
```bash
...
checkAppLocation
createNordVpnUserGroup
installHelperWithDependencies
...
clog "Done"
exit 0
```


Ignoring the second one, the first one is:

```bash
# workaround for rdar://33005768
checkAppLocation() {
    POSSIBLE_LOCATION="/Applications/NordVPN.localized/NordVPN.app"
    REQUIRED_LOCATION="/Applications/NordVPN.app"

    if [ -d $POSSIBLE_LOCATION -a ! -d $REQUIRED_LOCATION ]; then
      clog "Application installed in subfolder, moving it to /Applications"
      mv $POSSIBLE_LOCATION $REQUIRED_LOCATION
      rm -rf "/Applications/NordVPN.localized"
    else
      clog "Application available in /Applications"
    fi
}
```

It checks if `/Applications/NordVPN.localized/NordVPN.app` exists and `/Applications/NordVPN.app` does not exist. If true, the just installed (remember, we are inside the postinstall script...) NordVPN.app folder is moved to `/Applications` and the `NordVPN.localized` directory is removed; if false, everything is ok and a log message is printed.

The third function moves the binary helper to the proper directory, creates the launch daemon plist file, and loads it. Note that the property 'RunAtLoad' is set to true. It means that as soon as the plist is loaded with the launchctl utility, the binary is run.

```bash
installHelperWithDependencies() {

    ...

    clog "Installing Helper Tool"

    mkdir $HELPER_TOOL_DIR_PATH
    cp "/Applications/NordVPN.app/Contents/Library/LaunchServices/${HELPER_BUNDLE_ID}" "${HELPER_TOOL_DIR_PATH}${HELPER_BUNDLE_ID}"

    echo "<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<!DOCTYPE plist PUBLIC \"-//Apple//DTD PLIST 1.0//EN\" \"http://www.apple.com/DTDs/PropertyList-1.0.dtd\">
<plist version=\"1.0\">
<dict>
    <key>Label</key>
    <string>${HELPER_BUNDLE_ID}</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>MachServices</key>
    <dict>
        <key>${HELPER_BUNDLE_ID}</key>
        <true/>
    </dict>
    <key>Program</key>
    <string>${HELPER_TOOL_DIR_PATH}${HELPER_BUNDLE_ID}</string>
    <key>GroupName</key>
    <string>nordvpn_helper</string>
</dict>
</plist>" > "${LAUNCH_DAEMONS_DIR_PATH}${HELPER_BUNDLE_ID}.plist"

    chmod 700 "${LAUNCH_DAEMONS_DIR_PATH}${HELPER_BUNDLE_ID}.plist"

    launchctl load "${LAUNCH_DAEMONS_DIR_PATH}${HELPER_BUNDLE_ID}.plist"
}
```


Here, the trick to gain root privileges is described inside the first web page you get googling 'rdar://33005768' from the comment to checkAppLocation function ([here](https://openradar.appspot.com/33005768)):

```
Summary:
When installing an upgrade package that contains an application bundle that does not match the CFBundleIdentifier of the application that is being upgraded, macOS installer creates a .localized directory and installs the upgrade in this directory instead of upgrading the application in place.
```

Another important condition that is met is that the preinstall script is not properly written, so it does not completely remove files belonging to a possible previous installation.


## Exploit strategy

Linking together this last fact with what is described in the summary above, the attack could be summarized in these simple steps:

1. Create an application bundle inside `/Applications` called `NordVPN.app`, for example using: ```$ osacompile -o /Applications/NordVPN.app -e 'do shell script "uname -a > /tmp/uname"'```.
2. Modify the `Info.plist` of this new app, setting `CFBundleName` to `NordVPN`, `CFBundleIdentifier` to something different from `com.nordvpn.macos` and `CFBundleShortVersionString` to `8.14.6`.
3. Put a custom binary inside `/Applications/NordVPN.app/Contents/Library/LaunchServices/com.nordvpn.macos.helper` and make it executable.
4. Install the real NordVPN application with ```$ sudo installer -package NordVPN.pkg -target /```.


So what happens after 4. is: the preinstall script does not remove our `NordVPN.app` bundle, the check at `checkAppLocation` is false, so our bundle remains unmodified  and finally our custom privileged helper binary is moved and launched as root. BINGO!

The exploit is [here](https://github.com/p1tsi/misc/blob/main/NordVPN_LPE.sh).

## Remediation
The vendor has fixed this vulnerabilty by modifing the preinstall script, making sure that all files belonging to an eventual previous installation are removed.

```bash
...
removeApplicationsWithBundleID "com.nordvpn.NordVPN" 1
removeApplicationsWithBundleID "com.nordvpn.osx" 1
removeApplicationsWithBundleID "com.nordvpn.macos"
removeApp "/Applications/NordVPN.app"
...
```

In this way, they make sure that inside `/Applications` directory there are no files that could interfere on the installation process.



## Considerations
The attack scenario seems to be quite complex because a couple of preconditions should be met: the attacker should already have code execution as Administrator in order to prepare the environment and have write permissions to /Applications directory; the attacker should trick a real Administrator user to install NordVPN

## Conclusion

This flaw has been reported through HackeOne bug bounty program and the fix was released by the vendor on March 2024.

# Appendix

To manually extract `preinstall` and `postinstall` scripts and the whole application bundle, you can use the following script.

```bash
#!/bin/sh

# Check if the filename is provided
if [ $# -ne 1 ]; then
    echo "Usage: $0 <pkg_file>"
    exit 1
fi

PKG_FILE=$1

# Check if the file exists
if [ ! -f "$PKG_FILE" ]; then
    echo "File not found: $PKG_FILE"
    exit 1
fi

# Setup extraction folder
EXTRACTION_DIR=extraction
rm -rf $EXTRACTION_DIR
mkdir $EXTRACTION_DIR

# Extract .pkg file
xar -C $EXTRACTION_DIR -xf $PKG_FILE || exit

cd $EXTRACTION_DIR

# Get the value of tag 'pkg-ref' and remove the first character (it seems to be '#')
APP_DIR=`xmllint --xpath '//pkg-ref/text()' Distribution | cut -c2- `

cd $APP_DIR

# Extract preinstall and postinstall scripts if exist
if [ -f Scripts ]; then
    gunzip -dc "Scripts" | cpio -i
fi


# Extract Payload
gunzip -dc "Payload" | cpio -i

```