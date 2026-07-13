### Step 1:
Install the required packages
Session recording on Red Hat Enterprise Linux is enabled using the tlog package, and we will also be using the session-recording plugin for cockpit. These are found in the following two rpm packages, cockpit-session-recording and tlog.
```
dnf install -y cockpit-session-recording tlog
```
The first package, cockpit-session-recording will add an additional tab to Web Console which you will be using to enable and configure session recording. The tlog package will provide the tools which will be used to both record and view the recorded terminal sessions.
### Step 2:
Login to the Web Console
Now that the required software is installed, it is time to configure and enable terminal session recording. You will use the Web Console to perform this task.

Use the Web Console tab in your lab environment to log into the Web Console for the system. You should use the following credentials:

Username:

rhel

Password:

MjI2NTU3

Web Console Login

Figure 1. Web Console Login
<img width="1060" height="635" alt="image" src="https://github.com/user-attachments/assets/9669419c-3792-4cc8-b156-8d39fd87dfd8" />

NOTE: Problems accessing the Web Console or logging in? For best results, copy and paste the URL into Google Chrome.

Next, turn on administrative access. Click on the Turn on administrative access button.

Turn on administrative access

Figure 2. Turn on administrative access
<img width="1470" height="484" alt="image" src="https://github.com/user-attachments/assets/5a520754-b905-41cf-8deb-c1b286b3ccf1" />

Enter the Password redhat.

Enter the Password

Figure 3. Enter the Password
<img width="532" height="302" alt="image" src="https://github.com/user-attachments/assets/f5f91743-2588-48a8-90cc-ff6708ff42f7" />

Now that you are logged into the Web console, and because you have installed the cockpit-session-recording rpm package, you can now select the Session Recording option in the left-side navigation menu.

Session Recording

Figure 4. Session Recording
<img width="934" height="688" alt="image" src="https://github.com/user-attachments/assets/ccdd2df2-ba27-47d9-953e-13c8600922b9" />
### Step 3:
Enable session recording
The Session Recording application is initially blank, reporting No recorded sessions. Click the gear icon in the upper right-hand corner to be taken to the configuration settings for session recording.

Session Recording Configuration

Figure 1. Session Recording Configuration
<img width="1912" height="784" alt="image" src="https://github.com/user-attachments/assets/098c9f84-10e1-4f86-a894-0452f52387d2" />

On the resulting page, you will be offered access to configuration information for session recording. For the purpose of this scenario, you will accept most of the defaults and under the SSSD Configuration section at the bottom of the page, select the Scope of All, which will apply session recording to all users and groups that open sessions on the system.

All Scope Selected

Figure 2. Session Recording Configuration
<img width="1902" height="968" alt="image" src="https://github.com/user-attachments/assets/20f7fb88-06e8-4119-8f21-cbbbd247ce4a" />

Once you click the Save button, Web Console will write out a small configuration file detailing the scope for the sssd daemon.

After saving the configuration, return the Web Console to the main Session Recording page.

Return Main Session Page

Figure 3. Session Recording
<img width="1902" height="968" alt="image" src="https://github.com/user-attachments/assets/e67522f2-0542-4c0a-9ba6-71f79fb02f13" />
### Step 4:
Review the configuration

Switch back to the Terminal in your lab environment.

As mentioned on the previous step, the Web Console actions have written a small configuration file for sssd, /etc/sssd/conf.d/sssd-session-recording.conf You can review it to verify that the scope has been set to all so that all sessions for all users and groups will be recorded.
```
cat /etc/sssd/conf.d/sssd-session-recording.conf
```
Changes to the other configuration options displayed by Web Console would have stored those changes in /etc/tlog/tlog-rec-session.conf. For example, the notice message displayed to users who are having their session recorded. You can inspect this file as well, if desired.
```
cat /etc/tlog/tlog-rec-session.conf
```
### Step 5:
Record a session
Change user to the rhel user so that the session can be recorded.
```
su - rhel
```
You will notice that when the bash session starts, the rhel user receives the notice message configured in the tlog configuration.

Run some commands in the rhel user’s session.

You will need to copy and paste the following commands into the terminal in order for the recording to work peroperly. This is a limitation of the lab environment, not the tlog tool.
```
ls /tmp
who
df -hP
dnf list installed
```
Now that you have some data in a recorded session, you can log out of the user’s terminal session.
```
exit
```
### STep 6:
