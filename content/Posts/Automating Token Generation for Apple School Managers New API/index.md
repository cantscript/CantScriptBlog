+++
title = 'Automating Token Generation for Apple School Managers New API'
date = 2025-06-13
coverImg = "ASMAPIBanner.png"
summary = "I was looking at the new documentation when an article from Bart Reardon landed on my desk. He'd done the work already, so I went and automated the hell out of it"
tags = [ 'Workflows', 'API' ]
type = "blog"
+++

Like many of us, this week has been spent watching as many WWDC videos I can fit in as well as looking through all the new documentation.

One of the most antisipated sessions for us Apple Admins, or at least me, is the [What's New In Apple Device Management and Identity](https://developer.apple.com/videos/play/wwdc2025/258/?time=71) session. There was lots to be excited about but one thing that particularly got my attention was access to a new AxM API. 

I watched the video on the Tuesday morning _(since Im in the UK, Im not one to stay up all night watching these things)_ and before I even got to my desk messages were pinging around with links to newly released [AxM API Documentation](https://developer.apple.com/documentation/apple-school-and-business-manager-api)

Despite it not really being on my list of things to do that day, I went to have a look at it. With a quick scan I could see there was a section about how to [authenicate to gain access to the API](https://developer.apple.com/documentation/apple-school-and-business-manager-api/implementing-oauth-for-the-apple-school-and-business-manager-api). In a nutshell it looked like, copy and paste some things from AxM, create a token, use that token to create _another_ token and then use that to made the API calls. 

Theres an example python script there but as I said, it wasn't really on my list of things to do on that day. I thought to myself: _"I'll come back to it when I have some more time. Might be nice to figure out how to do all this with Bash though"_

Life went on. 

Wednesday rolled around and another message landed with a link for [Using the new API for Apple Business/School Manager](https://bartreardon.github.io/2025/06/11/using-the-new-api-for-apple-business-school-manager.html) by Bart Reardon. Bart had already turned the docs into plain english and with a step by step guide but even better, produced a `ZSH` script. On a side note, once I'd look at it I realised I was never going to figure out how to produce a bash script from the example phyton script....not without our good old friend ChatGTP.

So, this blog is **_NOT_** to give you a step by step guide on how this all works since Bart already did a great job with his blog. 

{{< callout circle-info >}}
So, if you haven't already, go read that and then come back and pick up here. 
{{< /callout >}}

Instead I want to talk about what I've done since seeing his `ZSH` script. At this point I can't decide if I've used smarts to automated the hell out of it....or over engineered a solution to a problem that isn't going to exist! In any case, I had fun with the little project so Im here to share. 


### Looking At Barts Script

As I mentioned, in a nutshell you have to generate a few tokens before you can even make an API call in the first place. Of course, this make things more secure but it also makes things more complicated. 

The first `token` you have to create is an `Client Assertion`, you then use this to generate a `Access Token` and with the `Access Token` you can finally make an API call. 

The `Client Assertion` is valid for 180 days (assuming it wasn't revoked in AxM by an Admin in the mean time). So once you've created this once, you haven't got to worry about it again for quite some time. 

The `Access Token` only lasts for an hour, which again isn't a bad thing from a security point of view. Once its expired you must create another one with a valid `Client Assertion`

While looking at Barts script you can see all of these pieces. The thing was the result of getting the `Client Assertion` was written out to a text file, you needed to go an copy paste that into variable into another script that is actually making the API call in order to create the `Access Token` which was stored locally in that script. 

So if I were running or testing multiple scripts I'd have to copy / paste that `Access Token` into many (since it seems wasteful to create a new one in each and every script if I had a valid one elsewhere). Then want happens when I can't actually remember which script has a valid `Access Token`, remember they are only valid for an hour. 

Even with just testing, nevermind writing something now and not coming to to it for weeks or months. I feel like I'd be forever generating that `Access Token`. For me this is taking away from the thing I want to spend more time working on, the AxM API and any automations I might want or need. 

The `Client Assertion` isn't really the issue here since its got a much longer expiration.

### Smart Automation or Over Engineered?

So I setout spliting out some of these componets to that all I ever needed to do in my AxM API scripts is add to lines of code at the top and it would take care of the `Access Token` management. 

If the `Access Token` didn't even exist, they'll check the `Client Assertion` is still valid and go ahead and create a new `Access Token`

If the `Access Token` exist, they'll check that its still valid _(ie, 60 mins old)_ but if not, they'll go ahead and create a a new `Access Token` providing there is a valid _(in date)_ `Client Assertion`

This way, wether its becuase I'm testing or if its becuase in production I only run this script every few months, I know my API calls will work. Further to this since its only 2 lines of code at the start of my script, its not bloated with logic and checks, again letting me just focus on the task at hand. 

As I say, its to been seen if this is even a problem we'll have. If it is, what I did to make the above a reality sways over to the **Over Engineered** side. 

### The Solution I Built

You can find **the project** over on my **GitHub** but lets take a few moment to see what I came up with

1. Built a Script that only deals with creating the `Client Assertion`
2. Saves the `Client Assertion` to a text file, along with a date/time stamp 180 days later
3. Built a second Script that only handle the creation of the `Access Token`
4. Saves the `Access Token` to a text file, along with a timestamp 60 mins later
5. Should an `Access Token` not exist, the second script will create an `Access Token` providing the  `Client Assertion` is still valid based on its date/time stamp 
6. Should there be an `Access Token` but its not valid based on its timestamp, the second script will create a new valid `Access Token`, again providing the `Client Assertion` is still valid based on its date/time
7. Enables two lines of code in the actual API script that creates/checks/renews the `Access Token` and saves the value into a variable for use in that script

This is all contained in a folder structure so as long as you add the scripts that interact with the API in the root of this folder, you only need to add the `values` you need from Axm to the _Create Client Assertion_ and _Create Access Token_ scripts once and no other variables are needed to make the automation work.  


### Configuring the Automation

First things first, if you haven't already go and read [Barts blog](https://bartreardon.github.io/2025/06/11/using-the-new-api-for-apple-business-school-manager.html) so that you know how to configure ASM. From ASM you'll need
* `The Private Key File` which will end in .pem <br>
* `Client ID` <br>
* `Key ID`

**Step 1** <br>
* Download the `AxM_API` folder from the [GitHub repo](https://github.com/cantscript/AxM_API).

It doesn't matter where this folder lives on the device as long as you know where you keep it as this is going to become the working folder for all of your ASM API scripts

**Step 2** <br>
Take your `Private Key File` and move it into the `AxM-API/AxMCert` folder

**Step 3** <br>
* Open `AxM-API/AutomationScript/create_client_assertion.sh` in a text/code editor <br>
* Enter the name of your `Private Key File` (so for example `myPrivateKey.pem`, not the location of the file) into the `private_key_file` variable <br>
* Enter your `Client ID` into the `client_id` variable <br>
* Enter your `Key ID` into the `key_id` variable <br>
* Save and close

**Step 4** <br>
* Open `AxM-API/AutomationScript/create_access_token.sh` in a text/code editor <br>
* Enter your `Client ID` into the `client_id` variable <br>
* Comment out either `scope="school.api"` or `scope="business.api"` depending on if you are interacting with ASM or ABM <br>
* Save and close

**Step 5** <br>
* Run `AxM-API/AutomationScript/create_client_assertion.sh`

### Using the Code

Any script that you want to use that interacts with the AxM API needs save to the root of the `AxM_API` folder. 

I've given a simple example script within the `AxM_API` folder. 

Your scripts just need the following two lines at the top

{{< codeimport url="https://raw.githubusercontent.com/cantscript/AxM_API/refs/heads/main/AxM_API/AsmApiCall.sh" type="bash" startLine=10 endLine=11 >}}

Then you will use the `accessToken` variable as the bearer token in a call. Below is a simple example. 

{{< codeimport url="https://raw.githubusercontent.com/cantscript/AxM_API/refs/heads/main/AxM_API/AsmApiCall.sh" type="bash" startLine=15 endLine=15 >}}

Notice that as part of the setup with didn't run the `create_access_token.sh`? Thats becuase on the first run of any script, the automation will see that there isn't one and will generate it on the fly for you.  

The next part is up to you! How you interact with AxM and the automations and workflow you create is actually the hard part and the part that gets the job done. 

If you haven't already seen, here are the [Apple Documents for the [ASM Endpoints](https://developer.apple.com/documentation/appleschoolmanagerapi) or [ABM Endpoints](https://developer.apple.com/documentation/applebusinessmanagerapi)

### A Quick Note

Although the scripts take care of keeping the `Access Token` valid, I didn't actually build in any "self renewal" of the `Client Assertion`. If this becomes invalid due to being over 180 days old, everything will just exit and error out. 

So if you need to renew this, run `AxM-API/AutomationScript/create_client_assertion.sh` again. 

_"You took the time to self renew the `Access Token` so why not the `Client Assertion`"_. Great Question! I just didn't, at least not today. Maybe next time I have a few minutes and I don't have a project Im not working on