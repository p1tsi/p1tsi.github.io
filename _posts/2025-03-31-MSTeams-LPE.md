---
title: "MacOS LPE via Microsoft Teams installation"
date: 2025-03-31 20:20:00 +0200
categories: [macOS, LPE]
tags: [post]
---


## Introduction

In this blogpost, I describe a vulnerability found in Microsoft Teams installation package. Its root cause is the same as the one I described in [NordVPN LPE post](https://p1tsi.github.io/posts/NordVPN-LPE/). So, I recommend reading it in order to have a more comprehensive description of the scenario.

## Root Cause Analysis

To sum up and keep this post as short as possible, I would only say that when distributing apps from vendor's website as `.pkg` files, if those packages contain `preinstall` and `postinstall` scripts, they should be well written and hardened so that eventual attackers having code execution as Administrator user are not able to exploit them to execute commands as root user.

Specifically, the first script should cleanup all files related to a previous version or currently installed instance of the same application inside `/Applications` folder.


In this case, if there is a previously installed `Microsoft Teams.app` (completely arbitrary bundle with this name) with a lower version and a different bundle id (for example `com.microsoft.teams3`), it won't be deleted by `preinstall` script. 

Here are the checks and cleanups that `preinstall` script performs (I report only some parts of the script. To have a complete view of it I suggest download the whole pkg from Microsoft's website and review it.):

1. Remove bundles with `com.microsoft.teams` id at `/Applications/Microsoft Teams.app`:
![Remove legacy app](assets/img/2025-03-31-MSTeams-LPE/remove_legacy_app.png)
2. Remove previous installed applications with `com.microsoft.teams2` id inside `/Applications/*.localized/` directory:
![Cleanup localized folder](assets/img/2025-03-31-MSTeams-LPE/cleanup_localized_teams.png)
3. Compare (eventually) installed bundle version and new version:
![Final Preinstall](assets/img/2025-03-31-MSTeams-LPE/final_preinstall.png)
with `check_for_downgrade`:
![Check for downgrade](assets/img/2025-03-31-MSTeams-LPE/check_for_downgrade.png)
If the attacker bundle version is less than the new one, `check_for_downgrade` function returns 1 and in shell scripting this means `false`. The execution of row 197, which creates a file that prevents `postinstall` script to be executed, is avoided. 

After the execution of `preinstall` script, the new `Microsoft Teams.app` bundle should be copied under `/Applications`, but, since there exists another one with the same name (and different bundle id, [read this](https://openradar.appspot.com/33005768)), macOS creates a `Microsoft Teams.localized` folder and puts the new bundle inside it.
So to sum up, before the execution of `postinstall` script, our `/Applications/Microsoft Teams.app` bundle is unremoved and unmodified.

To execute our custom code as root, it is necessary to reach a point inside `postinstall` script in which `/Applications/Microsoft Teams.app/Contents/Helpers/com.microsoft.teams2.migrationtool` (under attacker control) is launched. For example, inside the function `update_dock_icon`.

![Update dock icon function](assets/img/2025-03-31-MSTeams-LPE/run_migration_tool.png)

In the following check, the else is always executed because the `$INSTALLATION_TARGET_DIR` is `/Applications`.

![postinstall checks](assets/img/2025-03-31-MSTeams-LPE/postinstall_checks.png)

Now, one of `OLD_BUNDLE_PATH1` or `OLD_BUNDLE_PATH2` should be removed, so that the `update_dock_icon` is called.
For this reason, I created both bundles inside `/Applications`: they are removed and the check at row 170 is true. 
![Old bundle paths](assets/img/2025-03-31-MSTeams-LPE/old_bundle_paths.jpeg)
![Remove legacy apps](assets/img/2025-03-31-MSTeams-LPE/postinstall_remove_legacy_app.jpeg)


## Exploit Strategy

1. Clean `/Applications` folder from all Microsoft Teams stuff
2. Create an arbitrary application called `Microsoft Teams.app` having `MSTeams` as main binary inside `Microsoft Teams.app/Contents/MacOS`
3. Modify `Microsoft Teams.app/Contents/Info.plist` so that it contains an arbitrary `CFBundleIdentifier` and `24152.405.2925.6762` as `CFBundleVersion` and `CFBundleShortVersionString`
4. Create a binary with your arbitrary logic (this is what will be executed as root user) and put it inside `Microsoft Teams.app/Contents/Helpers/com.microsoft.teams2.migrationtool`
5. Create another app called `Microsoft Teams classic.app` and modify its `Info.plist` so that its `CFBundleIdentifier` is `com.microsoft.teams`
6. Create one or both of `Microsoft Teams (work or school).app` and `Microsoft Teams (work preview).app` bundles.
7. Install the real Microsoft Teams app opening its .pkg file

If a user unware of all this installs the real Teams app, he would insert its root password and during the installation, the custom `com.microsoft.teams2.migrationtool` binary will be executed as root.

The exploit is [here](https://github.com/p1tsi/misc/tree/main/MSTeams_LPE).


## Remediation

Waiting for Microsoft to fix the problem.


## Conclusion

I reported this issue to Microsoft through their Security Response Center and after a first triage, they classified it as moderate because of the need of Administrator privileges to properly prepare the environment for the exploitaiton.
As this product is not in their bug bounty program, they said my report is not eligible for bounties, CVEs or acknoledgements.
Since they were not able to tell me how long would it takes them to fix, I asked for the permissions to write and publish this post and they agreed.
