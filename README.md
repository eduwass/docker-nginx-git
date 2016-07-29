# docker-nginx-git
Minimal base image for hosting Nginx powered websites with Automatic Git Deployment functionalities including Webhooks

## Image Overview

| Component        | Why is it included? / Remarks |
| ---------------- | ------------------- |
| Ubuntu 16.04 LTS | The base system. |
| A **correct** init process | _Main article: [Docker and the PID 1 zombie reaping problem](http://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)._ <br><br>According to the Unix process model, [the init process](https://en.wikipedia.org/wiki/Init) -- PID 1 -- inherits all [orphaned child processes](https://en.wikipedia.org/wiki/Orphan_process) and must [reap them](https://en.wikipedia.org/wiki/Wait_(system_call)). Most Docker containers do not have an init process that does this correctly, and as a result their containers become filled with [zombie processes](https://en.wikipedia.org/wiki/Zombie_process) over time. <br><br>Furthermore, `docker stop` sends SIGTERM to the init process, which is then supposed to stop all services. Unfortunately most init systems don't do this correctly within Docker since they're built for hardware shutdowns instead. This causes processes to be hard killed with SIGKILL, which doesn't give them a chance to correctly deinitialize things. This can cause file corruption. <br><br>Baseimage-docker comes with an init process `/sbin/my_init` that performs both of these tasks correctly. |
| Fixes APT incompatibilities with Docker | See https://github.com/dotcloud/docker/issues/1024. |
| syslog-ng | A syslog daemon is necessary so that many services - including the kernel itself - can correctly log to /var/log/syslog. If no syslog daemon is running, a lot of important messages are silently swallowed. <br><br>Only listens locally. All syslog messages are forwarded to "docker logs". |
| logrotate | Rotates and compresses logs on a regular basis. |
| SSH server | Allows you to easily login to your container to [inspect or administer](#login_ssh) things. <br><br>_SSH is **disabled by default** and is only one of the methods provided by baseimage-docker for this purpose. The other method is through [docker exec](#login_docker_exec). SSH is also provided as an alternative because `docker exec` comes with several caveats._<br><br>Password and challenge-response authentication are disabled by default. Only key authentication is allowed. |
| cron | The cron daemon must be running for cron jobs to work. |
| [runit](http://smarden.org/runit/) | Replaces Ubuntu's Upstart. Used for service supervision and management. Much easier to use than SysV init and supports restarting daemons when they crash. Much easier to use and more lightweight than Upstart. |
| `setuser` | A tool for running a command as another user. Easier to use than `su`, has a smaller attack vector than `sudo`, and unlike `chpst` this tool sets `$HOME` correctly. Available as `/sbin/setuser`. |

Baseimage-docker is very lightweight: it only consumes 6 MB of memory.

## Configuration

### Available Configuration Parameters

The following flags are a list of all the currently supported options that can be changed by passing in the variables to docker with the -e flag.

 - **GIT_REPO** : URL to the repository containing your source code
 - **GIT_BRANCH** : Select a specific branch (optional)
 - **GIT_EMAIL** : Set your email for code pushing (required for git to work)
 - **GIT_NAME** : Set your name for code pushing (required for git to work)
 - **SSH_KEY** : Private SSH deploy key for your repository base64 encoded (requires write permissions for pushing)
 - **WEBROOT** : Change the default webroot directory from `/var/www/html` to your own setting
 - **ERRORS** : Set to 1 to display PHP Errors in the browser
 - **TEMPLATE_NGINX_HTML** : Enable by setting to 1 search and replace templating to happen on your code
 - **HIDE_NGINX_HEADERS** : Disable by setting to 0, default behaviour is to hide nginx + php version in headers
 - **PHP_MEM_LIMIT** : Set higher PHP memory limit, default is 128 Mb
 - **PHP_POST_MAX_SIZE** : Set a larger post_max_size, default is 100 Mb
 - **PHP_UPLOAD_MAX_FILESIZE** : Set a larger upload_max_filesize, default is 100 Mb
 - **DOMAIN** : Set domain name for Lets Encrypt scripts
 - **GIT_HOOK_TOKEN** : Auth-Token used for the [docker-hook](https://github.com/schickling/docker-hook) listener

### Dynamically Pulling code from git
One of the nice features of this container is its ability to pull code from a git repository with a couple of environmental variables passed at run time.

**Note:** You need to have your SSH key that you use with git to enable the deployment. I recommend using a special deploy key per project to minimise the risk.

### Preparing your SSH key
The container expects you pass it the __SSH_KEY__ variable with a **base64** encoded private key. First generate your key and then make sure to add it to github and give it write permissions if you want to be able to push code back out the container. Then run:
```
base64 -w 0 /path_to_your_key
```
**Note:** Copy the output be careful not to copy your prompt

To run the container and pull code simply specify the GIT_REPO URL including *git@* and then make sure you have also supplied your base64 version of your ssh deploy key:
```
sudo docker run -d -e 'GIT_REPO=git@git.ngd.io:ngineered/ngineered-website.git' -e 'SSH_KEY=BIG_LONG_BASE64_STRING_GOES_IN_HERE' richarvey/nginx-php-fpm
```
To pull a repository and specify a branch add the GIT_BRANCH environment variable:
```
sudo docker run -d -e 'GIT_REPO=git@git.ngd.io:ngineered/ngineered-website.git' -e 'GIT_BRANCH=stage' -e 'SSH_KEY=BIG_LONG_BASE64_STRING_GOES_IN_HERE' richarvey/nginx-php-fpm
```

## Docker-hook

`docker-hook` listens to incoming HTTP requests and triggers your specified command.

It is included in `/usr/bin/docker-hook` 

There is a listener script copied to `/usr/bin/hook-listener` that will auto-start


## Thanks to
* [phusion/baseimage-docker](https://github.com/phusion/baseimage-docker/) - Performance oriented base image
* [ngineered/nginx-php-fpm](https://github.com/ngineered/nginx-php-fpm) - Git push/pull functionalities
* [schickling/docker-hook](https://github.com/schickling/docker-hook) - Git Webhook listener
