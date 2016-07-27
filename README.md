# avakindle
Remote public warning display based on amazon kindle DX with OS version 2.5.8

For more information, visit: [snowtechblog.wordpress.com](http://snowtechblog.wordpress.com "What so ever")

## TODO'S
#### TODO: Add serverside script
#### TODO: list and explain futur improvements
#### TODO: Links, code formatting
#### TODO: Add start script, find it (on the kindle?)
#### TODO: add more comments to source code

## Source description:

The code is devided into three sections. The first block are function definitions. The second block are variables, initialize the WAN module and logfile, as well as defines that Kindle downloads the image on startup. The third block is the infinite while loop. OK, lets go through the third block to understand the logic ,-)

**Note:** You must set your username `$MAIL_USER`, passwort `$MAIL_PW`, email address `$MAIL_ADD` and recipient mail address `$MAIL_RECIPIENT` to get the `send_log` working.

#### Startup:

Open a terminal on Kindle (you should know how to do this, otherwise (LINK MOBILEREAD) is your friend...). Start the code with redirecting all possible output, cause this will write over the fullscreen image:
(FIND THE FILE ON KINDLE: START_LLB.SH or similar...)
Before entering the loop, function `WAIT_FOR_SS` will cause a wait until state screensave is reached (remove USB cable! slide power button once to go to screensave manually). At the startup the variable `$DOWNLOAD_IMG` will be set to `YES`, but lets neglect that for the moment, so its value is `NO`. So the code jumps into the function `WAIT_FOR_READY`.

#### WAIT_FOR_READY:

This is acually the heart of the loop, its doing much more than just wait. It waits until the state "Ready to suspend" and then calls the function to calculate the next wakeup (`calc_wakeup`) and to write the wakeup value to the RTC (`write_wakeup`).
The trick with the `$AWAKE_AGAIN` variable is to ensure that the function to calculate and to write the wakeup time is only called once and not multiple times (remember the state "ready to suspend" is 5 seconds long and I experienced that writing the wakeup time to often to the RTC can cause a failure and Kindle is in gone to never ending sleep). So always when Kindle is in a state where grep does not find anything with "Ready" (all other states), the `$AWAKE_AGAIN` is set to `YES` and as soon as entering "Ready" there are 5 seconds to perform `calc_wakeup` and `write_wakeup`.

#### CALC_WAKEUP: 

This checks the actions and calculates the next wakeup. There are three possibilities. 1st: The last download try failed (file not found on server, not connection etc.) and `$DL_FAILED` is `YES`, so Kindle will try in `$WAKEUP_CHECK_DEFAULT_FAIL` (def=600) seconds again. If `$DL_FAILED` is `NO`, we calculate the `$DIFF_TIME` which is the time between next action and the current time. If the `$DIFF_TIME` is larger than the `$WAKEUP_CHECK_DEFAULT` (def=3600) seconds, Kindle will wake up again in this time. If `$DIFF_TIME` is smaller, Kindle needs to consider to sleep again or to start the download. Note 1: the Kindle never wakes up excatly on time, so there is a time window (`$STAY_AWAKE` def=240sec) when to consider an action to be ready to download. Note 2: Better never try to sleep for less than 60 seconds (or 120 seconds?), remember that the wakeup process takes at least a minute in screensaver state.

In any case, the output of `$CALC_WAKEUP` will be a new value for the variable to trigger the download `$DOWNLOAD_IMG` and the next wakeup time `$WAKE_TIME`. The `$WAKE_TIME` will be used in `write_wakeup`:

#### WRITE_WAKEUP: 

Write the wakeup time to the RTC. First checks that `$WAKE_TIME` contains a reasonable value, that is `$WAKE_TIME` larger than `$WAKEUP_MINIMAL` (def=60sec) and that we have enough time to set the wakeup, that is `$TIME_LEFT` in "Ready for Suspend" is larger than `$LATEST_WAKEUP_SET` (def=0sec, we have to risk everything to try to write the wakeup ...). If Kindle should download the image (this is done in Active state), we dont want to write a wakeup (could be in conflict with switching the state) but bring the kindle into active (NOTE: this should not be done by `powerd_test -p` without any state checking, cause this can make a lot of trouble... definitely a point to improve!).

OK, now Kindle wakes up `$WAKEUP_CHECK_DEFAULT` (every hour) and on specific times defined in `$ACTION` (space separated string of time, in my case two actions per day always at 08:15 and 17:15). Now, when an Action is active, variable `$DOWNLOAD_IMG` is set to `YES` and the conditional in the infinite loop is entered. Remember that we enter in Screensave mode (hopefully), so `wait_for_ss` will not delay. Then, we switch into active to have 10minute to perform all the stuff we want Kindle to do in the action. In my case its defined in `DOWNLOAD_LLB`.

#### DOWNLOAD_LLB: 

Turns WAN on, checks for connection, download image, sync the time and sends the logfile via email. If its not possible to get connection, `set_retries` is called to count the number of retries. To prevent from checking infinitely long every ten minutes for a download, the retries are limited to the number `$DL_RETRIES` (def=3). This results in `$DL_FAILED=YES` for the next run of `calc_wakeup`. If Kindle is able to connect, the task mentioned are performed. For the download server set the variable $IMG_URL to your needs. Note that Kindles version of wget does not support to check for a never file. This would be a nice-to-have. So a second improvement goes into that direction ...


#### SEND_LOG: 
`send_log` need some special notes. Kindle does not give a mail agent, so I use sendmail from a recent busybox compilation. I found that a precompiled ARM7L version works on kindle (LINK???). You need to specify the path to it in the function. And you need to setup the sendmail command to work with your provide, e.g. set the variables `$MAIL_USER`, `$MAIL_PW`, `$MAIL_ADD` and `$MAIL_RECIPIENT` to your values.

#### DISPLAY_IMAGE:

Last and least, we need to display the downloaded image with `display_image`. This is done with Kindles internal command `eips`. Since my project should work in the cold, Kindle runs `eips` several times in hope that the eInk display gets better contrast by writing the pixels several times.
