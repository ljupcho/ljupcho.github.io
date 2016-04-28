---
layout: post
title:  "Create application in python2.7 with django-nonrel and mongodb on openshift"
date:   2013-08-21 21:18:10 +0200
tags: python mongodb django
comments: true
---	


1. Create an account on openshift 

2. Create app by choosing python2.7. You need not to choose any git repository or set other settings just choose “create application”. After that's done, you can switch to your localhost and create a folder with your app_name and execute those commands that you get upon creation of the app. That will let you make modification locally and just “git push” your changes to the cloud.

3. Add cartridge mongodb to your app and save those credentials.

4. Before going into the code you need to install some pre-requirements on the cloud. Login with ssh that you're provided with for your app on openshift and ssh to it. Go to tmp folder with: cd $OPENSHIFT_TMP_DIR and install first pip and then do these:<br/>
	<code>
	# pip install https://bitbucket.org/wkornewald/django-nonrel/get/tip.tar.gz<br/>
	# pip install https://bitbucket.org/wkornewald/djangotoolbox/get/tip.tar.gz<br/>
	# pip install https://github.com/django-nonrel/mongodb-engine/tarball/master
	</code><br/>
You can install these with pip, but what I did was “wget” all of these then “tar xcvf” them, cd to each of the extracted folders and install them with “python setup.py install”. After installation you can verify by typing “python”, then in the shell “import django”, “django.get_version()”, and output should be “1.3”. At this point you have python and django-nonrel set up. Moving on.

5. What you need to do now is modify the sample code generated from openshift and you have it locally after “git clone”, you can create yourself a py project in Eclipse and import it. Now, you can get this code (https://github.com/ljupcho/openshift_python2.7_django-nonrel_mongodb) and copy the folders and files, change the folder name 'blog' to whatever your app_name is and simple git push from terminal from your local app folder. 

6. That should actually do it, you should have the app now running, since the "git push" restart the application as well.

7. You will also need to sync the database, so log in with ssh to application and do “cd ~/app-root/repo/blog” and execute: “python manage.py syncdb” With this you can add superuser that you can log in to admin django with. 

8. You can also buy yourself a domain and link it to the rhcloud app.

