# django_ansible

Example repository for a tutorial on how to deploy Django with Ansible


In this tutorial I will explain how to deploy a Django project in 15 minutes with Ansible. I will assume that you are a Django developer and you have built and tested a project locally. It’s time to deploy the project on a public server to let users access your awesome application.

If you are new in deploying Django on a production server you can read my post Django – NGINX: deploy your Django project on a production server to have a basic introduction on the steps needed.

So you need a VPS with an SSH access, then you will access the server, install and configure all necessary software (web server, application server, database server), create a database user, configure Django to use it, copy your Django project on the server, migrate the database, collect static files, trial and error, fix, trial and error, …

All this boring stuff will take some good hours that you should definitely spend in a more profitable way, don’t you think? The good news is that you can automate almost all the work needed to go from a vanilla VPS to a fully deployed server hosting your Django project.

Follow this tutorial and I’ll show you how to leverage the power of Ansible to automate all the needed steps in 15 minutes. Are you ready? Check the time on your clock and follow me!

## Setup the SSH access to your VPS

### Add the VPS address to your SSH configuration

If you use OpenSSH client on Linux/UNIX, you can add an entry like this in ~/.ssh/config:

```markdown
Host yourserver
User root
Port 22
HostName yourserver.example.com
```

Pay attention to what you use on Host value, yourserver in this example, because this is the label you’ll use in the following steps to refer to this particular server.

In HostName you’ll configure the actual IP address or hostname of your VPS, according to the access data received by your provider.

### Configure SSH access without using a password

To be able to connect to your VPS without using a password, you have to setup a public/private SSH key on your workstation (if you don’t already have one), this is very simple and can be done with the following command:

```
ssh-keygen
```
Then you can configure a password-less access to your VPS using the command:

ssh-copy-id yourserver
That’s it. You should now be able to connect to your VPS using SSH without entering your password every time.

# 2. Deploy your Django project with Ansible

## Clone the template repository

I prepared a template repository of a Django project, you can clone it at the following address before proceeding:

git clone <https://github.com/baxeico/django_ansible.git>
In the repository you’ll find three directories:

## django
This directory is a sample Django project created with the usual:

django-admin.py startproject yourproject
I renamed the root directory created by the command to be simply django.

It is just a bare template not meant to be used as is, but it should give you an idea on how the Django settings are supposed to be structured in this tutorial.

I always follow a best practice when starting a new Django project, by splitting the monolithic settings.py file in different files, one for each deploy environment (local, test, production, …). You can read a more in depth explanation of this approach if you aren’t used to it.

So inside yourproject sub-directory you’ll find a settings package containing different modules for different deploy environments:

* base.py – This is the common settings which will be inherited (and possibly overridden) from all deploy environments;
* local.py – This will be the DJANGO_SETTINGS_MODULE used in your local development environment. Nothing special here, it only makes Django to use sqlite as DB for local development;
* production.py – This will be the DJANGO_SETTINGS_MODULE used in production. Here you’ll find that the database used by Django is defined using dj-database-url package, by means of an environment variable. Also the STATIC_ROOT settings variable is defined by means of an environment variable.

## requirements

This directory contains the minimum Python requirements of your project, you’ll find three files there:

* base.txt – contains the basic requirements for all deploy environment (only Django in the example, it’s a Django project after all, isn’t it?);
* local.txt – it inherits from base.txt and adds Ansible as a requirement for your local development environment. Version 2.3.0 is the latest Ansible version at the time of writing;
* production.txt – it also inherits from base.txt and adds dj-database-url as a requirement for your production environment. It will be used to configure the database used by Django by means of the environment variable DATABASE_URL.
Notice how this structure mimics the different Django settings modules described above. You can install the local requirements using this command:

```bash
pip install -r requirements/local.txt
```

I strongly suggest to use Virtualenv, together with virtualenvwrapper to create and manage an isolated environment for your project.

ansible
In this directory you’ll find the most interesting part, that is a set of Ansible playbooks to automate the installation and configuration of the server and the deployment of your project.

I suggest to copy the ansible directory in your project root, and eventually adapt the playbooks to your needs.

The first thing to customize is the hosts file. This is the file where you could list all the hosts controlled by Ansible. In this example you should have only one entry, corresponding to the Host value you entered in SSH client configuration.

Then you can proceed to rename the file host_vars/yourserver, using the label you gave to your server in the hosts file. Here you’ll also find some variables used in the playbooks.

Here is a brief description of each playbook you’ll find:

config_files.yaml – Copy nginx and uwsgi configuration files on remote server.
deploy.yaml – Deploy your Django project on the server, pulling the master branch from your GIT repository, installing all needed production requirements, running migrate and collectstatic and restarting uwsgi.
packages.yaml – Install needed software packages on remote server using apt.
postgresql.yaml – Create and configure the access to a Postgresql database used by your Django project on remote server. Notice how the database password is randomly generated and stored in the local file postgresqlpasswd. The password will be automatically inserted in the DATABASE_URL environment variable, to be used in Django production settings.
system.yaml – Create an user ubuntu on remote server, together with a private/public SSH key pair. The public key is returned as output when you run the playbook, to be used as a “deploy key” on the server. More details later on this step.
upgrade.yaml – Upgrade all apt packages on remote server.
Push your Django project in a publicly accessible GIT repository
Probably you already have your project sources hosted on a public GIT repository like Github or Bitbucket. If you don’t, this is a perfect time to do it! I personally use Bitbucket, because it offers an unlimited number of private repositories.

In the following steps I assume that you have your Django project hosted on Bitbucket, under this path:

ssh://git@bitbucket.org/youruser/yourproject.git
Let the Ansible automation begins!
Inside the ansible directory, run the following command:

./ansible.sh system.yaml
You should see an output similar to this:

ok: [yourserver] => {
 "changed": false,
 "msg": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1kvkW9... ansible-generated on yourserver.example.com\n"
}
What you find in the msg variable is the public SSH key generated for the ubuntu user on remote server. You should copy the public key and add it as a “deploy key” in the settings of your GIT repository.

A deploy key is a read-only SSH key that will be used to clone your repository from the remote server. You can find more details in the Bitbucket and Github documentation.

Now you are ready to complete the deploy of your Django project! Run the following commands inside the ansible directory:

./ansible.sh packages.yaml
./ansible.sh postgresql.yaml
./ansible.sh config_files.yaml
./ansible.sh deploy.yaml
If all goes well you should be able to reach your Django project on your remote server public address, on port 80.

3. Update your Django project
Keeping your Django project updated on the remote server is very easy in this setup. You only need to push your changes to your GIT repository (on the master branch) and then you can run the following command inside the ansible directory:

./ansible.sh deploy.yaml
This playbook will perform the following tasks for you:

pull the updated master branch from your GIT repository;
eventually install new Python production requirements, and update the existing ones;
migrate the database to the latest version;
update your static files with collectstatic;
restart uwsgi.
4. Continuous delivery using Bitbucket Pipelines
In the follow up article Bitbucket Pipelines and Ansible: Continuous delivery for your Django project I explained how you can use Ansible together with Bitbucket Pipelines to implement a simple yet powerful continuous delivery setup for your Django project.

5. Conclusions
In this article I explained how to deploy a Django project in 15 minutes with Ansible. The key here is in following some Django best practices and leveraging the power on Ansible to quickly deploy a Django project on a remote server. I think that the time spent to learn and master those technologies is worth the effort for the time you’ll save in the future and all the human errors you’ll avoid by automating and standardizing your processes.

Of course this is only the tip of the iceberg of what you could do and automate with Ansible, but I found this simple approach to be already a big time saver in my daily work.

Have fun with Ansible!
