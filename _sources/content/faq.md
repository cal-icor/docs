# Frequently Asked Questions

## Can I see my students servers?
Yes! You need to navigate to Admin section of the server.
1. Navigate to the Admin Section:
    - From a notebook: File > Hub Control Panel;
    - From a url: [your_server].jupyter.cal-icor.org/hub/admin (e.g. csumb.jupyter.cal-icor.org/hub/admin)
2. Search for the User Name in Search box
3. Click Start Server (You may have to refresh the page)
4. Click Access Server
5. You will prompted to click an orange button asking for authorization to use your account to login in the user's server.

* Be careful! Best Practice is to then logout so you do not mistakenly start working in the student files.

## R-Studio is giving a 403 error when it tries to save
This is most likely happening when a users has had over thirty minutes of inactivity; the system shuts the server down but 
user interface still looks like a session is active. 

A student may close their laptop, move on for a time period, then open their laptop,
see the R-Studio interface and start "working". The work will not be saved and 403 errors will start to pop up in modal boxes as R Studio attempts to automatically save.

The user should refresh the browser tab every time they begin to work in R Studio if they have been away from the computer for more than 30 minutes.

## Can I add packages for R or Python?
Yes. If you add the packages to your VM via pip install or the R package manager they will disappear between sessions.

If you would like the packages to be permanently available for all users on your hub, please fill out the Package Request (GitHub issue)[https://github.com/cal-icor/cal-icor-hubs/issues/new?template=package_request.yml]. In most cases, we should be able to install the desired package(s).

If you have a lot of packages you need installed you can fill a normal (GitHub Issue)[https://github.com/cal-icor/cal-icor-hubs/issues/new?template=BLANK_ISSUE] instead of package-by-package in the Package Request issue template.
