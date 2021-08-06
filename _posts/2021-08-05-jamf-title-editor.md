# Introducing... JAMF Title Editor!

Remember [my last post about patching 3rd party applications with Kinobi](https://automateordie.io/using-kinobi-open-source-to-patch-custom-lob-applications/)? Well, JAMF has implemented that natively into the JAMF Pro interface [as of version 10.31.0](https://docs.jamf.com/10.31.0/jamf-pro/release-notes/New_Features_and_Enhancements.html#concept-1673) (released 2021-07-29). So now we don't even need Kinobi!

### The good news 🎉
The interface and terminology more or less resemble Kinobi. If you've been creating your own patch definitions with Kinobi, you'll have no problem moving to JAMF. The process is almost the exact same.
You no longer need a separate Kinobi server either!

### The bad news 😫
There's no way to export your patch titles/definitions from Kinobi Open-Source so you'll have to recreate them from scratch in JAMF. It should be a little easier this time since you already have the right data & values, but I won't lie - it's gonna be tedious, especically if you have a lot of titles.
However, if you have your definitions saved as JSON files, JAMF's Title Editor has an "Import JSON" button for you to use. Lucky you!

#### Alright already, just show me how to use this new feature!
[Read up on JAMF's excellent documentation](https://docs.jamf.com/title-editor/documentation/Setting_Up_Title_Editor_in_Jamf_Pro.html).

#### Final thoughts...
Of course once I spend days setting up Kinobi for our company and then _more_ days writing up a blog post on how to use it, JAMF decides to integrate the product into their solution and effectively neuter any reason for using Kinobi Open-Source. I'm not really complaining though - it's a totally logical move and I'm happy to have one less VM I have to keep updated. Plus, there was no guarantee that Mondada would ensure Kinobi Open-Source would work with future versions of JAMF. At least with this move, I _know_ my custom title definitions will always work.
