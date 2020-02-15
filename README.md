# nc-test-01-puppet-nginx
First test for NC


## Worklog
2020-02-15 - 11:00 to 11:15h started repository on GitHub and started some basic documentation
2020-02-15 - 11:15 to 12:15h Setup basic Puppet Server, an Agent and pdk in Docker containers. No local setup for now.
2020-02-15 - 12:15 to 12:30h Publish some basic setup results and raw module. Finish this iteration. It's weekend after all! 


## Test goals
This is the test info received through email:
```text
Create/extend an existing puppet module for Nginx including the following functionalities:

Create a proxy to redirect requests for https://domain.com to 10.10.10.10 and redirect requests for https://domain.com/resource to 20.20.20.20.
Create a forward proxy to log HTTP requests going from the internal network to the Internet including: request protocol, remote IP and time take to serve the request.
(Optional) Implement a proxy health check.
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
pdk new task && echo $?
```
