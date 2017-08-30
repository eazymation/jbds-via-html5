This repository is forked from, and largely based upon, the work from
Richard Lucente's github repository at rlucente-se-jboss/jbds-via-html5.
Much of this Readme is borrowed from the Readme  of that repository also.
That work describes how to get  jboss developer studio running in a container
and accessable via the web using apache guacamole. The goal of this repository 
is to run eclipse sirius instead. So minor differences are that Eclipse Sirius is
started instead of jboss developer studio (they are both eclipse based projects
which run in a JVM). Also I try to ensure that this can
be run on openshift online as well as a local openshift 'minishift' platform
(openshift online does not allow the do a strategy=docker deploy, 
which is only a minor hurdle , also openshift online has a 10 project limit).
We use the Obeo Designer install as it neatly packages sirius componenents with 
miminal set of plugins.


## Introduction
[Apache Guacamole](https://guacamole.incubator.apache.org/) is an
incubating Apache project that enables X window applications to be
exposed via HTML5 and accessed via a browser.  This article shows
how guacamole can be run inside containers in an OpenShift Container
Platform (OCP) cluster to enable eclipse sirius , an
eclipse-based IDE for modelling applications, to be accessed
via a web browser.  

## How does Apache Guacamole work?
Apache Guacamole consists of two main components, the guacamole web
application (known as the guacamole-client) and the guacamole daemon
(or guacd).  An X windows application runs in an Xvnc environment
with an in-memory only display.  The guacd daemon acts as an Xvnc
client, consuming the Xvnc events and sending them to the Tomcat
guacamole-client web application where they are then rendered to
the client browser as HTML5.  Guacamole also supports user
authentication, multiple sessions, and other features that this
article only touches on.  The [Apache Guacamole](https://guacamole.incubator.apache.org) web site has
more information.

## Login to OpenShift Container Platform
This was tested on a cloud-based OpenShift installation as well as
a laptop using the [Red Hat Container Development
Kit](https://developers.redhat.com/products/cdk/overview/).  This
article uses CDK 3, so please adjust accordingly if using an
alternative OpenShift Container Platform installation.  CDK3 leverages
the `minishift` command to stand up a virtual machine for OCP.

From a command line terminal, configure and start minishift:

    minishift setup-cdk --default-vm-driver virtualbox 
    minishift start --cpus 4 --disk-size 50g --memory 10240 --username 'RHN_USERNAME' --password 'RHN_PASSWORD'

It is possible that the first step will not be required if you have 
already setup the minishift environment, and possible that you will
be using a different default-vm-driver such as KVM.

Substitute the `RHN_USERNAME` and `RHN_PASSWORD` credentials above
with your login credentials from either the
[Red Hat Developer's Portal](https://developers.redhat.com) or
the [Red Hat Customer Service Portal](https://access.redhat.com).
You also may need to change the command line flags or set additional
command line flags for your environment.  You can see all the options
for the `minishift` commands by adding the `--help` option.

Make sure to add the `oc` command to your executable search path.
On my laptop, the path is `$HOME/.minishift/cache/oc/v3.5.5.8/oc`.
Use whatever path is appropriate for your minishift installation.

Once minishift has finished starting up, determine the IP address
for the minishift instance then login:

    IP_ADDR=$(minishift ip)
    oc login https://$IP_ADDR:8443 -u developer

For minishift, the password is 'developer'.

## Enabling Unprivileged Guacamole Client Containers
The guacamole project supplies Docker Hub images to simplify deploying
guacamole in a container.  However, the guacamole-client runs as a
privileged container by default.  A thin wrapper around the guacamole
image was created so it could run unprivileged within OpenShift.
Please refer to the
[guacamole-client-wrapper](https://github.com/rlucente-se-jboss/guacamole-client-wrapper)
project on github for more information on how this was done.  That
project was used to extend the `guacamole/guacamole` image on Docker
Hub to create the `rlucentesejboss/guacamole` image that is used
for the guacamole-client.

## Install the Guacamole Components
First, create a project for guacamole within the OpenShift Container
Platform.

    oc new-project guacamole

A persistent MySQL instance stores guacamole data including users
and their credentials.  Create the guacamole MySQL instance and
then modify it to use a persistent volume.  The MySQL database
persists users and connection parameters within guacamole.

    oc new-app mysql MYSQL_USER=guacamole MYSQL_PASSWORD=guacamole \
        MYSQL_DATABASE=guacamole
    oc volume dc/mysql --add --name=mysql-volume-1 -t pvc \
        --claim-name=mysql-data --claim-size=1G --overwrite

The guacamole image includes helper scripts for database initialization.
Run the guacamole image to create a database initialization script
for the MySQL database.  Use the `oc run` command to run the image
with an alternative start command.

    oc run guacamole --image=rlucentesejboss/guacamole --restart=Never \
        --command -- /opt/guacamole/bin/initdb.sh --mysql 

The `initdb.sh` command runs within a pod named `guacamole`.  When
the command completes, the MySQL initialization script will be in
the container log.  Put the initialization script into a SQL file
and remove the pod.

    oc logs guacamole > initdb.sql
    oc delete pod guacamole

At this point, the MySQL pod should be fully running, but it may
have restarted due to the deployment configuration change to add
the persistent volume claim.  Get the list of running pods to
determine the pod-id for MySQL.

    oc get pods

Identify the path to the MySQL client application within the pod.
To do that, type the following:

    oc rsh mysql-<pod-id>
    echo $PATH | cut -d: -f1
    exit

Use the pod-id and the executable path from the above command to
initialize the guacamole database:

    oc rsh mysql-<pod id> <exec-path>/mysql -h 127.0.0.1 -P 3306 \
        -u guacamole -pguacamole guacamole < initdb.sql

The above line initializes the MySQL database with all of the tables
and artifacts required to support guacamole.  Once the database is
initialized, create an application where both guacamole and guacd
are in a single pod.  The additional parameters will connect guacamole
to its database.

    oc new-app rlucentesejboss/guacamole+guacamole/guacd \
        --name=holy \
        GUACAMOLE_HOME=/home/guacamole/.guacamole \
        GUACD_HOSTNAME=127.0.0.1 \
        GUACD_PORT=4822 \
        MYSQL_HOSTNAME=mysql.guacamole.svc.cluster.local \
        MYSQL_PORT=3306 \
        MYSQL_DATABASE=guacamole \
        MYSQL_USER=guacamole \
        MYSQL_PASSWORD=guacamole

The last thing to do is expose a route for the guacamole application.

    oc expose service holy --port=8080 --path=/guacamole

## Configure Guacamole Users
Browse the the guacamole application.  On the CDK, the URL is
[holy-guacamole.192.168.42.66.nip.io/guacamole](holy-guacamole.192.168.42.66.nip.io/guacamole).
Make sure that the URL is appropriate for your environment. You can
find the exact URL for your environment by going to the openshift web
console, opening the guacamole project, and clicking overview. You will
see the URL in the top right corner :

![Guacamole Overview Screen](images/guacamole_route.png)
The
login page for guacamole will appear.  Use the default username and
password of `guacadmin/guacadmin` as shown.

![Guacamole Login Screen](images/admin-login.png)

Once logged in, go to the upper right hand corner and select
"guacadmin -> Settings" in the drop down menu, as shown.

![Guacamole Settings](images/admin-settings.png)

Select the "Users" tab and then click the "New User" button.

![Guacamole Users](images/admin-users.png)

Set the username and password to whatever you desire.  As an
administrator, you can create multiple user accounts that can use
guacamole to connect to their own instances of Sirius.
Also, grant the permissions "Create new connections" and
"Change own password".  Click "Save" to add the user.

![Guacamole Add User](images/admin-add-user.png)
![Guacamole Created User](images/admin-created-user.png)

Log out of the guacamole web application.

![Guacamole Logout](images/admin-logout.png)

The user is now configured to create a connection to their instance
of Sirius and access it via a browser.  

## Sirius Container build
We have already built  the Sirius container image and stored it in dockerhub.
It downlopaded and used the Obeo Designer 10 install.
Also in that container we have added bundles to allow the user to easily use
the  example family model. Note that to do this we created an eclipse update site,
which is stored as the directory. 
https://github.com/neilmackenzie/jbds-via-html5/tree/master/resources/family_updatesite

To create that update site we followed the instructions here:
https://www.eclipse.org/forums/index.php/t/1076701/

You can see how this conatiner is built by looking at the docker file in this repository.
We reference this from dockerhub which builds and stores the container:
https://hub.docker.com/r/neilmackenzie/jbds-via-html5/


## Instantiate the Sirius Container
Each user simply provisions a Sirius container instance and then
grants guacamole permission to view it. If using openshift online 
we suggest just adding this app to the same project, since only 10 projects
are allowed in openshift online. in this example we create a seperate project.
You will not need to grant access if you are deploying in the same project.

Execute the commands below:

    oc new-project someproject
    oc policy add-role-to-user view system:serviceaccount:guacamole:default
    oc new-app neilmackenzie/jbds-via-html5 --name=sirius 
   
    
If the apps are deployed in the same project we suggest using 
'oc new-app neilmackenzie/jbds-via-html5 --name= sirius-X' 
where sirius-X is sirius-1 or sirius-2 etc (we will deploy one sirius app per user).

## Access the Sirius  Container via a Browser
A developer can now access the sirius application
via a browser.  On the CDK, the URL is
[holy-guacamole.192.168.42.66.nip.io/guacamole](holy-guacamole.192.168.42.66.nip.io/guacamole).
Make sure that the URL is appropriate for your environment.  When
presented with the login screen, use the username/password that was
created by the guacamole administrator.  Once logged in, in the upper right hand corner
select "username -> Settings", as shown.

![Guacamole User Settings](images/user-settings.png)

Select the "Connections" tab and then click the "New Connection"
button.

![Guacamole New Connection](images/user-new-connection.png)

Set the following parameters:

| Parameter | Value |
| --------- | ----- |
| Name | sirius |
| Hostname | sirius.someproject.svc.cluster.local |
| Port | 5901 |
| Password | VNCPASS |

Click "Save" to add the connection. Hostname is the name of the service, 
it will depend upon the projects and app name that you chose (like sirius or sirius-X)
It can be viewed by looking at the service in the openshift console

![Guacamole Connection Settings](images/user-connection-settings.png)

In the upper right hand corner, select "username -> sirius" to open
the connection.

![Guacamole Sirius Connection](images/user-jbds-connection.png)

Obeo Developer Studio will appear within the browser window.

![Guacamole Sirius](images/user-sirius.png)

## Try it out

click new -? example- sirous family model, then new representation and play with the model.

