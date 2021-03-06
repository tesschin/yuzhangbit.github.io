---
published: true
title: Automatically host a gollum wiki with the supervisor
classes: wide
categories:
  - Tools
tags:
  - Automation
---

Gollum wiki plays an important role in my research and study for keeping small pieces of knowledge organized. When I want to write a note, or keep some useful links, I fire up a web browser and write them down in my wiki through the built-in web editor for markdown. Thus, hosting my wiki locally all the time is pretty important for me.

Previously, I use service tools (such as systemd, upstart) coming with the Ubuntu system to automatically host the wiki when system starts.
But sometimes you are not even sure which tool you system is using. Maybe both exist. And it seems not that simple to set up a service for a beginner.

Thus, I switch to the supervisor, a process control system, to manage the programs running in the background. So far, I am happy with it since it's easy to install, configure and update. In addition, it provides a web gui to control the state of programs. What a nice feature to have!

## Dependencies
* [rbenv]() (ruby version manager, avoid messing up with the ruby comes with the system)
* [bundler]() (ruby application gem manaer, install and update gems with ease)
* [gollum]()  (wiki engine)
* [supervisor]() (Automatically host the wiki when the system starts, also provide a nice gui to control programs)


## Installation
Here is a simple gollum wiki template: [https://github.com/yuzhangbit/wiki_barebone.git](https://github.com/yuzhangbit/wiki_barebone.git) that we are going to use.

Run the install script in the repo. This install script has only been tested on ubuntu 14.04 LTS and 16.04 LTS.
```bash
git clone https://github.com/yuzhangbit/wiki_barebone.git
cd wiki_barebone  
```
If it is your first time to install dependencies, please run commands below.  
```bash
bash install.bash  # install dependencies, rbenv, bundler, gollum, supervisor, enable the web gui for supervisor
bash setup.bash    # set up the autostart configuration for the wiki app
```    

If you have already installed the dependencies, make your wiki automatically start using commands below,
```bash
bash setup.bash
```
## Usage
Open your browser and check the wiki out.
```bash
localhost:8888
```
#### Start and Stop wiki
This wiki will be hosted automatically when you start the ubuntu. You can control the program through commands below or web gui interfaces.
```bash
sudo supervisorctl start wiki     # start to host the wiki, the "wiki" is defined by the APP_NAME variable.
sudo supervisorctl restart wiki   # restart to host the wiki, the "wiki" is defined by the APP_NAME variable.
sudo supervisorctl stop wiki      # stop to host the wiki,  the "wiki" is defined by the APP_NAME variable.
```

### Supervisor
**NOTE**: If you install `supervisor` via `pip`, you probably can not generate a configuration file `/etc/supervisor/supervisord.conf` and the `/etc/supervisor/conf.d` folder. Even if you create the configuration file and directory for it manually, you are going to have trouble with `sockets` when running `supervisorctl` commands. So the `apt-get install` method is recommended.


## Configuration
In order to manage all `programs`, `supervisor` creates a `/etc/supervisor/conf.d` directory to hold all the `.conf` files for all the programs. In its `supervisord.conf` file, it defines
```
[include]
files = /etc/supervisor/conf.d/*.conf
```
folder to find all the `.conf` files.  If this is missing, please add it manually.

An example `.conf` file can be generated by `setup.bash` script in [https://github.com/yuzhangbit/wiki_barebone](https://github.com/yuzhangbit/wiki_barebone).


### Enable the web gui
The web gui tool for the supervisor can be enabled in `/etc/supervisor/supervisord.conf` by adding
```
[inet_http_server]
port = 127.0.0.1:9001
```
to `/etc/supervisor/supervisord.conf`. This is done by the `install.bash` script in [https://github.com/yuzhangbit/wiki_barebone](https://github.com/yuzhangbit/wiki_barebone)
Then you can check the status of programs managed by the supervisor through `localhost:9001` in web browser.

### Useful Commands for Supervisor
```bash
sudo supervisorctl reload  # reload the supervisor.conf file and restart supervisor
sudo supervisorctl reread # reread the .conf files for programs
sudo supervisorctl udpate # update all the programs
sudo supervisorctl start all # start all the programs
```
