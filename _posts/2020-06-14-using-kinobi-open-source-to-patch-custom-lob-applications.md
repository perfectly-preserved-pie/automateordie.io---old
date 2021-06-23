# Context
We use JAMF at our company but had a problem with keeping our line-of-business (LOB) applications updated. [JAMF's Patch Management catalog](https://docs.jamf.com/jamf-app-catalog/Patch_Management_Software_Titles.html), while extensive, doesn't include applications hiding behind a company's paywall: VPN clients, NAC agents, AV/EDR agents, etc. 
In our case, one of those applications is Palo Alto GlobalProtect. While the application has its own auto-upgrade mechanism, I'm using it here as an example of what you can do with Kinobi Open-Source.

# External patch sources
Aside from JAMF's internal catalog (which is free), JAMF also allows you to configure an external patch source, which is really just a [webserver that can respond to specific API requests](https://www.jamf.com/jamf-nation/articles/497/jamf-pro-external-patch-source-endpoints). That JAMF article explains the API and JSON structure needed for a functioning external patch source. An external patch source will allow you to create a software title, definitons, etc. for _any_ application and therefore let you update _any_ application; you're not just limited to whatever's in the JAMF catalog. The downside is that _you_ have to define the application, its requirements, its versions, etc. (and keep them updated!).
To get a hint of what you _can_ (but sometimes don't need) to define, take a look at [Kinobi's page on the topic](https://mondada.atlassian.net/wiki/spaces/MSD/pages/553189450/Patch+Definitions).

# Introducing Kinobi Open-Source (not self-hosted, not cloud)
That JAMF article is pretty intimidating if you've never worked with APIs or JSON in general. Thankfully, we can stand on the shoulder of giants and use the solutions created by people have already done the impressive work before us. One of these solutions is Kinobi Open-Source.
Kinobi is an external patch definition server that comes in 3 different flavors:
* Kinobi Cloud, a cloud-hosted patch definition server for Jamf Pro with access to a Kinobi subscription and JSON importer.
* Kinobi Self-Hosted, a self-hosted patch definition server for Jamf Pro with access to a Kinobi subscription and JSON importer.
* Kinobi Open-Source, an open-source patch definition server for Jamf Pro.

The first two (Cloud and Self-Hosted) require a paid subscription to use. The last one, Open-Source, is truly free and is what we'll be using here.

# Installing Kinobi Open-Source
## System Requirements
Installation is pretty simple; the installer is a .run script that can run on 
* Ubuntu LTS Server 14.04 or later (18.04 recommended)
* Red Hat Enterprise Linux (RHEL) 6.4 or later
* CentOS 6.4 or later

See the full system requirements [here](https://github.com/mondada/kinobi#standalone).

## Actually installing it
[There's already an installation guide provided by Mondada so I'll just link it here.](https://mondada.atlassian.net/wiki/spaces/MSD/pages/592216069/Kinobi+Open-Source)

### Don't install Kinobi on the same server hosting your JAMF Pro instance! (if you're not using JAMF Cloud)
If you install Kinobi on the JAMF Pro server, Kinobi won't start; it uses port 443 which is already in use on a JAMF Pro server so the port bind will fail. You can check the status of Kinobi's Apache webserver with `sudo systemctl status apache2.service`.
You'll see an error message similiar to "Bind: Address Already in Use".

For that reason, it's best to spin up a new VM/server and install only Kinobi on that.

# How to manually add a software title
And now we get to the meat of it 🍖
Mondada has [an excellent guide](https://mondada.atlassian.net/wiki/spaces/MSD/pages/553222153/Manual+Creation) on how to create your first software title.
In my case, I'll be creating one for Palo Alto GlobalProtect.

** GlobalProtect example
Click on the "New" button to start the process, then fill out the information specified.
![Starting the process with some basic information](https://i.imgur.com/1u6dsQy.png)
* **Name**: this shows up in the JAMF GUI.
* **Publisher**: this shows up in the JAMF GUI.
* **Application Name**: optional. Skip it.
* **Bundle Identifier**: optional. Skip it.
* **Current Version**: What's the latest version of this software you have in your environment? Put that here. It'll be used in the patch reports as the latest version.
* **ID**: This is just an internal reference for Kinobi. You can make it anything you want but I tend to just put the application name.

When you're finished, click Save. You've just created your first software title! 🎉 Now we need to add a "requirement".

## What's a Requirement (for a software title)?
Simply put, a requirement (per Kinobi) is "Criteria used to determine which computers in your environment have this software title installed."
The syntax, form, and structure is exactly the same as a smart group or advanced search in JAMF. Ever created one of those? Maybe you've created a smart group or advanced search based on things like
* ARM or x86_64 CPU architecture
* Presence of an installed application
* Operating System Version
* Extension Attribute value
* etc.

It's the same process for our software title requirement. You can make the requirement any of a number of different criteria but in general, *your requirement should be the easiest way to detect the presence of the application on a computer*; that'll usually be "Application Title". 
So go ahead and click the "Add" button to begin creating a requirement.
Click the drop-down menu and select "Application Title" and click OK.
![Requirements](https://i.imgur.com/mlJ5g8g.png)

Then change the **operator** to "is" and the **value** to "GlobalProtect.app"
![Criteria](https://i.imgur.com/XOWuSvI.png)




## Patch definitions for each app versions are needed otherwise they all show up as "Unknown"

* Next blog post about using PatchCLI to autogenerate the JSON for easy importing into Kinobi
