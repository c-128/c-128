# 17.02.24: MusterHackd, or how I hacked my school as an 11th grader
One day I was sitting bored in my room and was in search of a new project. I wanted to do some more bug hunting [after successfully bypassing the Veyon screen lock](./veyonunlocked.md). So I had a look at the services my school provides. [Moodle](https://moodle.org/), [SOPHOS](https://www.sophos.com/) and [linuxmuster](https://linuxmuster.net/). All the typical stuff you would expect from a school. However, linuxmuster stood out particularly. I knew that it was very important. It is used to provide a SAMBA server, manage the PC's operating systems (images) and other critical services. Therefore I decided to go down another bug hunting journey. This time linuxmuster was my target. 

First of all, you want to get an idea of what you are working with. Therefore I logged into my school's linuxmuster server with a webapp. At my school you could only see a few details about your account, change your password and access the files in your home drive. This was pretty simple, but I was pretty sure that the web app had a lot more features, thus I started to look for some documentation. To my surprise linuxmuster was completely open source. Looking at the [Github organization](https://github.com/linuxmuster/) shows that the project is split up in 3 main parts:
- [Sophomorix 4](https://github.com/linuxmuster/sophomorix4) as a SAMBA 4 server implementation.
- [Linbo 7](https://github.com/linuxmuster/linuxmuster-linbo7) as an image management tool to install/update the image on a single PC.
- [WebUI 7](https://github.com/linuxmuster/linuxmuster-webui7) as a webapp to control the Sophomorix 4 server and a general server management tool, using the [Ajenti](https://github.com/ajenti/) framework.

At this point I was pretty sure that I had logged into the WebUI. But before browsing the source code, I opened up my browser's devtools to inspect the HTTP requests.

After logging out and logging back in, the application sends a request to a login endpoint. Upon successful login, this endpoint responds with a `Set-Cookie` header to update the `session` cookie. Linuxmuster uses a typical session-based [(?)](https://roadmap.sh/guides/session-based-authentication) authentication.
```http
POST /api/core/auth
Content-Type: application/json

{
   "username": "<username>",
   "password": "<password>",
   "mode": "normal"
}
```
The application continued to send requests, such as getting the currently logged in user, session timeout and other requests. However, one stood out, the `/api/lmn/quota/user/<username>` endpoint. This endpoint returned a lot of personal information, such as initial password in clear text, birthday, first and last name,... I was curious why the `username` parameter was needed since the server already knew from which user the request came from due to the `session` cookie, therefore I tried another username in the URL parameter. And to my surprise it gave me a response with all the details, even for teachers.

Digging further, I looked at the requests for the home drive access endpoints. I noticed that there was one endpoint to list all the files for a directory.
```http
POST /api/lmn/smbclient/list
Cookie: session=<session_cookie>
Content-Type: application/json

{
   "path":"\\\\server\\default-school\\students\\<class>\\<username>/<folder>"
}
```
My immediate thought was what if it joined the path [(?)](https://www.geeksforgeeks.org/python-os-path-join-method/) in an unsafe way, allowing you to add `../` to go up the directory tree, up to the root of the server's filesystem. No matter how many escapes I added, it only gave me the directories I actually had access to, and it didn't even get to the root filesystem.

After finding this out, I decided to get a first look of the source code. At the time of finding these exploits, the latest WebUI 7 commit was [`157e01d641`](https://github.com/linuxmuster/linuxmuster-webui7/tree/157e01d641c48b8eb0e2667ed10b5265ee793d68) and the latest Ajenti commit was [`c9561232a7`](https://github.com/ajenti/ajenti/tree/c9561232a7c20fa75f6bb267231b41b0ed3f9ded/).  
I started browsing the two repositories, starting with the Ajenti framework. Both are written using JS for the frontend, and Python for the backend. The framework linuxmuster uses, Ajenti has a flexible structure, where every feature set is a plugin. Every plugin provides a set of REST endpoints as Python functions annotated with `@get('<path>')` or `@post('<path>')`. In addition to providing simple endpoint annotations, Ajenti also takes care of the frontend using Angular. Each plugin provides it's own Angular module which the browser loads. In addition to these features, the framework also provides a permissions API. Permissions are very simple as they are only a string. Users are assigned a mapping of permission and truth value. The truth value is true if they have the permission, false if not. Endpoints are annotated using `@authorize('<permission>')` to check if the requesting user has the permission.  
The linuxmuster web interface is basically just a set of plugins for the Ajenti framework. You can tell them apart by their `lmn_` prefix. These plugins provide linuxmuster related endpoints, such as Linbo or SAMBA integration.

With this knowledge, I started to investigate the linuxmuster endpoints. Many of the things shown in the WebUI are just a subprocess hidden behind a REST interface. My idea was to escape the subprocess using things like `$(sleep 10)`, `` `sleep 10` `` or the backspace character (`\b`). After trying this on many endpoints, I couldn't get anything to work.

This failure did not stop me. While looking for endpoints that use a subprocess I noticed that only some endpoints have an `@authorize()` annotation. I decided that this must mean that these endpoints do not check the user's permissions.  
The first endpoints I noticed were in the [`lmn_common`](https://github.com/linuxmuster/linuxmuster-webui7/blob/157e01d641c48b8eb0e2667ed10b5265ee793d68/usr/lib/linuxmuster-webui/plugins/lmn_common/) plugin. This plugin provides no features for the user, but it serves as a library for the other linuxmuster plugins. The plugin provides two noticeable endpoints, one being the [`/api/lmn/diff`](https://github.com/linuxmuster/linuxmuster-webui7/blob/157e01d641c48b8eb0e2667ed10b5265ee793d68/usr/lib/linuxmuster-webui/plugins/lmn_common/views.py#L157) endpoint and the other being [`/api/lmn/log/<path>`](https://github.com/linuxmuster/linuxmuster-webui7/blob/157e01d641c48b8eb0e2667ed10b5265ee793d68/usr/lib/linuxmuster-webui/plugins/lmn_common/views.py#L27). These did not check the user's permission, but worse yet, there was a comment that has been sitting there for 5 years, `TODO authorize`. These endpoints are both very powerful, as they will read any file from the server given the file path, as long as it was a text file. I didn't have any file paths I could try, except `/etc/passwd` and other standard linux files. The `/etc/passwd` [(?)](https://linuxize.com/post/etc-passwd-file/) file is important as it contains a list of all the users on the linux server. I also tried to read the `/etc/shadow` [(?)](https://linuxize.com/post/etc-shadow-file/) file, but I got a "permission denied" error. This makes sense since each session is running in an isolated user context without root [(?)](https://www.howtogeek.com/737563/what-is-root-on-linux/) privileges.
```http
### Read files using the log endpoint, like "cat /etc/passwd"
GET /api/lmn/log//etc/passwd
Cookie: session=<session_cookie>

### Read files using the diff endpoint, like "diff /etc/passwd /dev/null"
POST /api/lmn/diff
Cookie: session=<session_cookie>
Content-Type: application/json

{
   "file1": "/etc/passwd",
   "file2": "/dev/null"
}
```
And to no surprise I got a response from both endpoints with all linux users on the server.

Digging deeper into the endpoints, I looked at some plugins Ajenti provides. One of them was the [`session_list`](https://github.com/ajenti/ajenti/blob/c9561232a7c20fa75f6bb267231b41b0ed3f9ded/plugins/session_list) plugin. It is very simple as it's purpose was to provide a list of all authenticated user sessions on the Web UI. It provides a single [`/api/session_list/sessions`](https://github.com/ajenti/ajenti/blob/c9561232a7c20fa75f6bb267231b41b0ed3f9ded/plugins/session_list/views.py#L21) GET Endpoint. Unfortunately, it doesn't check user permissions. It responds with an array of all active sessions, including the session's `session` cookie. With this cookie you can take over any user that is currently logged in. For example, if an admin is logged in when making the request, you can take that `session` cookie, paste it into your browser's cookie devtools and you are now the admin user. You can now do whatever you want, delete all the files, leak sensitive information and much more.

Before I end this post, I would like to give a huge shout out to the linuxmuster team for taking the bugs seriously and fixing the exploits within 8 hours of reporting them and quickly rolling out updates to schools within the next 48 hours.

Thanks for taking them time to read this! If you have any questions left, feel free to open an issue in this repository.  
[Check out my other posts](../README.md)
