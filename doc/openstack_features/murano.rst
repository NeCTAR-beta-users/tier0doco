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




