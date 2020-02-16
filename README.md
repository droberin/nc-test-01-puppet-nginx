# nc-test-01-puppet-nginx
First test for NC


## Worklog
* 2020-02-15 - 11:00 to 11:15h started repository on GitHub and started some basic documentation
* 2020-02-15 - 11:15 to 12:15h Setup basic Puppet Server, an Agent and pdk in Docker containers. No local setup for now.
* 2020-02-15 - 12:15 to 12:30h Publish some basic setup results and raw module. Finish this iteration. It's weekend after all! 
* 2020-02-15 - 13:20 to 14:00h Friends being late... let's add some modules to puppet, like nginx and prepare target servers to be forwarded to ;)
* 2020-02-15 - 14:00 to 14:20h Friends still not here. Add info for testing nginx from servers
* 2020-02-16 - 14:00 to 14:30h Try creating a file inside an Ubuntu puppet agent running in Docker
* 2020-02-16 - 14:30 to 15:00h Deploy simple nginx in a client
* 2020-02-16 - 15:00 to 15:30h A man must eat
* 2020-02-16 - 15:30 to 17:00h Deploy final nginx with redirections and test them
* 2020-02-16 - 17:00 to 17:20h Update README, upload code, have a break...



## Test goals
This is the test info received through email:
```text
Create/extend an existing puppet module for Nginx including the following functionalities:

Create a proxy to redirect requests for https://domain.com to 10.10.10.10 and redirect requests for https://domain.com/resource to 20.20.20.20.
Create a forward proxy to log HTTP requests going from the internal network to the Internet including: request protocol, remote IP and time take to serve the request.
(Optional) Implement a proxy health check.
```

## Prepare target goals
```bash
docker network create --internal --subnet 10.10.10.0/24 nginx-net-10
docker network create --internal --subnet 20.20.20.0/24 nginx-net-20
docker run -d --name nginx-10 --net nginx-net-10 --ip 10.10.10.10 nginx:alpine
docker exec -ti nginx-10 ip addr | awk '{if($7~"eth0"){print $2"\n"}}
10.10.10.10/24
docker run -d --name nginx-20 --net nginx-net-20 --ip 20.20.20.20 nginx:alpine
docker exec -ti nginx-20 ip addr | awk '{if($7~"eth0"){print $2"\n"}}
20.20.20.20/24
```
None of these 2 servers are reachable through external network.

### Test nginx from target servers inside their own networks
```bash
docker pull curlimages/curl
docker run --rm -ti --net nginx-net-10 curlimages/curl curl -I 10.10.10.10 > /dev/null; echo $?
docker run --rm -ti --net nginx-net-20 curlimages/curl curl -I 20.20.20.20 > /dev/null; echo $?
```

## Deploying Puppet
### Server and a basic agent
I just deployed [Puppet Server in a docker container|https://puppet.com/blog/puppet-docker-running-puppet-container-centric-infrastructure/] as they say in their doc, also a single agent.

```bash
docker network create puppet
docker run --net puppet --name puppet --hostname puppet puppet/puppetserver-standalone
docker run --net puppet puppet/puppet-agent-alpine
```
### PDK may be helpful too
#### Grab image
```bash
docker image pull puppet/pdk
```
#### Interact with pdk
Let's make interactions with pdk a little bit easier. This docker image seems to be using `/root` as `WORKDIR` and `pdk` command as `entrypoing`, plus adds some scripts to `/root`, so let's keep them, just in case.

```bash
export NC_TEST_WD=$PWD
mkdir -p modules/nginx_manager_nc
docker run --rm -v ${NC_TEST_WD}/modules/nginx_manager_nc:/root/nginx_manager_nc -ti puppet/pdk new module"
```

Let's get some output
```text
pdk (INFO): Creating new module: 

We need to create the metadata.json file for this module, so we're going to ask you 5 questions.
If the question is not applicable to this module, accept the default option shown after each question. You can modify any answers at any time by manually updating the metadata.json file.

[Q 1/5] If you have a name for your module, add it here.
This is the name that will be associated with your module, it should be relevant to the modules content.
--> nginx_manager_nc

[Q 2/5] If you have a Puppet Forge username, add it here.
We can use this to upload your module to the Forge when it's complete.
--> username

[Q 3/5] Who wrote this module?
This is used to credit the module's author.
--> Roberto Salgado

[Q 4/5] What license does this module code fall under?
This should be an identifier from https://spdx.org/licenses/. Common values are "Apache-2.0", "MIT", or "proprietary".
--> MIT

[Q 5/5] What operating systems does this module support?
Use the up and down keys to move between the choices, space to select and enter to continue.
--> Debian based Linux

Metadata will be generated based on this information, continue? Yes
pdk (INFO): Using the default template-url and template-ref.
pdk (INFO): Module 'nginx_manager_nc' generated at path '/root/nginx_manager_nc'.
pdk (INFO): In your module directory, add classes with the 'pdk new class' command.
```

Let's see what we've got in here
```bash
$ ls ${NC_TEST_WD}/modules/nginx_manager_nc
pdk-module-target20200215-6-9g07fc
```

So let's use this module as work dir for `pdk` command.
```
alias pdk="docker run --rm -v ${NC_TEST_WD}/modules/nginx_manager_nc:/root/nginx_manager_nc -w /root/nginx_manager_nc/pdk-module-target20200215-6-9g07fc puppet/pdk"
```
Now we can test if the module is found by pdk inside container:
```bash
pdk new task
```
__Hint__: When this command doesn't find a valid directory it will warn you instead of providing help due to missing parameters. Just testing this alias, no task will be created.

#### Alias puppet and install Nginx Module to puppet
Set a quick alias for puppet
```
alias puppet="docker exec -ti puppet puppet"
```
And install this module
```bash
puppet module install puppet-nginx
```
Seems it works fine
```text
Notice: Preparing to install into /etc/puppetlabs/code/environments/production/modules ...
Notice: Downloading from https://forgeapi.puppet.com ...
Notice: Installing -- do not interrupt ...
/etc/puppetlabs/code/environments/production/modules
└─┬ puppet-nginx (v1.1.0)
  └─┬ puppetlabs-concat (v6.2.0)
    ├── puppetlabs-stdlib (v6.2.0)
    └── puppetlabs-translate (v2.1.0)
```

## Get an Ubuntu based puppet agent
```bash
docker run --net puppet --net nginx-servers --name nginx-ubuntu --hostname nginx-ubuntu puppet/puppet-agent-ubuntu
```

## Test puppet is working
* Create a simple «pp» file with a simple file to ensure a file exists or will be created inside a node called `nignx-ubuntu-test`.
```bash
cat > ${NC_TEST_WD}/nginx_manager_nc.pp <<EOF
File { backup => false }

node nginx-ubuntu-test {
  file { '/tmp/puppet-in-docker':
    ensure  => present,
    content => 'This file is just a probe that we managed to reach this puppet client inside Dockah.',
  }
}
EOF
```
* As no volume was on deployment time for puppet, let's just copy this file into the containers storage in a dirty way.

```bash
docker cp ${NC_TEST_WD}/nginx_manager_nc.pp puppet:/etc/puppetlabs/code/environments/production/manifests/nginx_manager_nc.pp
```
* Run an agent matching `hostname` requirement
```bash
docker run -ti --net puppet --hostname nginx-ubuntu-test puppet/puppet-agent-ubuntu
```

* See result in stdout
```text
Notice: /Stage[main]/Main/Node[nginx-ubuntu-test]/File[/tmp/puppet-in-docker]/ensure: defined content as '{md5}ca592fcd274ce5330e43d195b30f2301'
```

## Test deploying basic nginx service
* Install `apt` module
```bash
puppet module install puppetlabs-apt
```

* Modify existing pp file and copy it to master. We'll remove all the garbage created later

```bash
cat > ${NC_TEST_WD}/nginx_manager_nc.pp <<EOF
include nginx

File { backup => false }

node domain.com {
  nginx::resource::server { 'domain.com':
    listen_port => 80,
    proxy       => 'http://10.10.10.10:80',
  }
}
EOF
docker cp ${NC_TEST_WD}/nginx_manager_nc.pp puppet:/etc/puppetlabs/code/environments/production/manifests/nginx_manager_nc.pp
```

Notice node is now called `domain.com`.

* Now let's run a container with a puppet agent image using bash entrypoint so we can interact with it and check without it dying.

```bash
docker run -ti \
 --name nginx-ubuntu-domain-com \
 --net puppet \
 --hostname domain.com --entrypoint /bin/bash \
 puppet/puppet-agent-ubuntu
```

* From another terminal connect it to target networks
(I use byobu for multiplexing non X server dependant)
```bash
docker network connect nginx-net-10 nginx-ubuntu-domain-com  
docker network connect nginx-net-20 nginx-ubuntu-domain-com  
```

* This image has no `ip` nor `ifconfig` commands so lets just go with curl into the servers to check connectivity from docker container terminal
```bash
curl -I 10.10.10.10 ; curl -I 20.20.20.20
```

* Since we ain't using image's entrypoint, no puppet agent has been run in this container. Original entrypoint seems to be:
```text
ENTRYPOINT ["/opt/puppetlabs/bin/puppet"]
CMD ["agent", "--verbose", "--onetime", "--no-daemonize", "--summarize"]
```

* puppet's set in a dir in PATH, so let's run it
```bash
puppet agent --verbose --onetime --no-daemonize --summarize
```

* This fails due to the lack of apt packages cache, so, let's update it manually even if puppet can do it for you.
and run the agent again.
```bash
apt-get update
puppet agent --verbose --onetime --no-daemonize --summarize
```

* Let's check output
```text
Notice: /Stage[main]/Apt/Package[gnupg]/ensure: created                                                                                                
Notice: /Stage[main]/Nginx::Package::Debian/Apt::Source[nginx]/Apt::Key[Add key: 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 from Apt::Source nginx]/Apt_k
ey[Add key: 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 from Apt::Source nginx]/ensure: created                                                           
Notice: /Stage[main]/Nginx::Package::Debian/Apt::Source[nginx]/Apt::Setting[list-nginx]/File[/etc/apt/sources.list.d/nginx.list]/ensure: defined conten
t as '{md5}9e145555a5e8343dd79b4e77f61dc112'
Info: /Stage[main]/Nginx::Package::Debian/Apt::Source[nginx]/Apt::Setting[list-nginx]/File[/etc/apt/sources.list.d/nginx.list]: Scheduling refresh of $
lass[Apt::Update]    
Info: Class[Apt::Update]: Scheduling refresh of Exec[apt_update]
Notice: /Stage[main]/Apt::Update/Exec[apt_update]: Triggered 'refresh' from 1 event
Notice: /Stage[main]/Nginx::Package::Debian/Package[nginx]/ensure: created
Info: Class[Nginx::Package]: Scheduling refresh of Class[Nginx::Service]
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/conf.stream.d]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/conf.mail.d]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/run/nginx]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/snippets]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/var/log/nginx]/group: group changed 'root' to 'adm'
Notice: /Stage[main]/Nginx::Config/File[/run/nginx/client_body_temp]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/run/nginx/proxy_temp]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/sites-available]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/sites-enabled]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/streams-enabled]/ensure: created
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/streams-available]/ensure: created
Info: Computing checksum on file /etc/nginx/nginx.conf
Info: /Stage[main]/Nginx::Config/File[/etc/nginx/nginx.conf]: Filebucketed /etc/nginx/nginx.conf to puppet with sum f7984934bd6cab883e1f33d5129834bb
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/nginx.conf]/content: content changed '{md5}f7984934bd6cab883e1f33d5129834bb' to '{md5}4de5b329de5965
8456477800a1dd1bd5'
Info: Computing checksum on file /etc/nginx/mime.types
Info: /Stage[main]/Nginx::Config/File[/etc/nginx/mime.types]: Filebucketed /etc/nginx/mime.types to puppet with sum cbd0fe7ed116af80a5a81b570559146e
Notice: /Stage[main]/Nginx::Config/File[/etc/nginx/mime.types]/content: content changed '{md5}cbd0fe7ed116af80a5a81b570559146e' to '{md5}b43779580ce373
ff33648aa888424046'
Info: Class[Nginx::Config]: Scheduling refresh of Class[Nginx::Service]
Notice: /Stage[main]/Main/Node[domain.com]/Nginx::Resource::Server[domain.com]/Concat[/etc/nginx/sites-available/domain.com.conf]/File[/etc/nginx/sites
-available/domain.com.conf]/ensure: defined content as '{md5}1c2a9a32c0d3a0e81173b1ea4ce6ce00'
Info: Concat[/etc/nginx/sites-available/domain.com.conf]: Scheduling refresh of Class[Nginx::Service]
Notice: /Stage[main]/Main/Node[domain.com]/Nginx::Resource::Server[domain.com]/File[domain.com.conf symlink]/ensure: created
Info: /Stage[main]/Main/Node[domain.com]/Nginx::Resource::Server[domain.com]/File[domain.com.conf symlink]: Scheduling refresh of Class[Nginx::Service]
Info: Class[Nginx::Service]: Scheduling refresh of Service[nginx]
Notice: /Stage[main]/Nginx::Service/Service[nginx]/ensure: ensure changed 'stopped' to 'running'
Info: /Stage[main]/Nginx::Service/Service[nginx]: Unscheduling refresh on Service[nginx]
Notice: Applied catalog in 14.01 seconds
```

* Alright, now see if nginx is running.
```text
# pidof nginx
1574 1572 1571 1570 1569
```

* Check access to server, since it will recognise itself as domain.com we can curl it:
```bash
# curl -I domain.com
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Sun, 16 Feb 2020 15:08:40 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Tue, 21 Jan 2020 14:39:00 GMT
ETag: "5e270d04-264"
Accept-Ranges: bytes
```

* Let's look for redirection
```bash
# cat /etc/nginx/sites-available/domain.com.conf 
# MANAGED BY PUPPET
server {
  listen *:80;


  server_name           domain.com;


  index  index.html index.htm index.php;
  access_log            /var/log/nginx/domain.com.access.log combined;
  error_log             /var/log/nginx/domain.com.error.log;

  location / {
    proxy_pass            http://10.10.10.10:80;
    proxy_read_timeout    90s;
    proxy_connect_timeout 90s;
    proxy_send_timeout    90s;
    proxy_set_header      Host $host;
    proxy_set_header      X-Real-IP $remote_addr;
    proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header      Proxy "";
  }
}
```

* Let's replace index of nginx-10 and nginx-20 to easily see if it's actually responding
```bash
docker exec -ti nginx-10 sh -c 'echo I am nginx-10 > /usr/share/nginx/html/index.html'
docker exec -ti nginx-20 sh -c 'mkdir  /usr/share/nginx/html/resource ; echo I am nginx-20 resource! > /usr/share/nginx/html/resource/index.html'
```

* Curl into domain.com from agent container again to see if we can reach nginx-10
```bash
# curl domain.com
I am nginx-10
```

* Good but we don't have any redirection for the `resource` context of `domain.com`. Let's try fixing that from first terminal.
```bash
cat > ${NC_TEST_WD}/nginx_manager_nc.pp <<EOF
include nginx

File { backup => false }

node domain.com {
  nginx::resource::server { 'domain.com':
    listen_port => 80,
    proxy       => 'http://10.10.10.10:80',
  }
  nginx::resource::location { "/resource":
    server      => 'domain.com',
    proxy       => 'http://20.20.20.20:80',
  }
}
EOF
docker cp ${NC_TEST_WD}/nginx_manager_nc.pp puppet:/etc/puppetlabs/code/environments/production/manifests/nginx_manager_nc.pp
```

* Run agent again in agent console
```bash
puppet agent --verbose --onetime --no-daemonize --summarize
```

* Now let's check both paths again
```bash
root@domain:/# echo Root: $(curl domain.com 2>/dev/null) ; echo Resource: $(curl domain.com/resource/ 2>/dev/null)
Root: I am nginx-10
Resource: I am nginx-20 resource!
```

* Finally, let's check into nginx configuration
```bash
root@domain:/# cat /etc/nginx/sites-available/domain.com.conf 
# MANAGED BY PUPPET
server {
  listen *:80;


  server_name           domain.com;


  index  index.html index.htm index.php;
  access_log            /var/log/nginx/domain.com.access.log combined;
  error_log             /var/log/nginx/domain.com.error.log;

  location / {
    proxy_pass            http://10.10.10.10:80;
    proxy_read_timeout    90s;
    proxy_connect_timeout 90s;
    proxy_send_timeout    90s;
    proxy_set_header      Host $host;
    proxy_set_header      X-Real-IP $remote_addr;
    proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header      Proxy "";
  }

  location /resource {
    proxy_pass            http://20.20.20.20:80;
    proxy_read_timeout    90s;
    proxy_connect_timeout 90s;
    proxy_send_timeout    90s;
    proxy_set_header      Host $host;
    proxy_set_header      X-Real-IP $remote_addr;
    proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header      Proxy "";
  }
}
```

## TODO: Get logs into remote graylog server
```bash
cat > ${NC_TEST_WD}/nginx_manager_nc.pp <<EOF
include nginx

File { backup => false }

node domain.com {
  nginx::resource::server { 'domain.com':
    listen_port => 80,
    proxy       => 'http://10.10.10.10:80',
    log_format  => 'graylog2_format  \'$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" \';'
    access_log  => 'syslog:server=graylog2.example.org:12301 graylog2_format'
    error_log   => 'syslog:server=graylog2.example.org:12302'
  }
  nginx::resource::location { "/resource":
    server      => 'domain.com',
    proxy       => 'http://20.20.20.20:80',
  }
}
EOF
docker cp ${NC_TEST_WD}/nginx_manager_nc.pp puppet:/etc/puppetlabs/code/environments/production/manifests/nginx_manager_nc.pp
```


## TODO
* Send logs into graylog...
* Convert this into a module as we created it before
* Proxy healthcheck (optional) I think nowadays one uses the ones that are provided by the platform, like healthcheck and liveprobeness from Kubernetes anyway... so if service is not working, platform will fix that instead.

