# Instructions

Here are the steps that I took to deploy a Django Web Application to AWS Elastic Beanstalk using an RDS managed database running PostgreSQL.  


## Prerequisites
Pre-requisites: Python, PostgreSQL, pip, virtualenv, awsebcli (with aws credentials set up) 

Here are some links to help if you need to install any of these. You will also need to make sure you have an AWS account and your credentials set up for the AWS CLI. Instructions for that can be found here: AWS CLI Setup 

Python Install Instructions 
pip Install Instructions 
virtualenv Install Instructions 
awsebcli Install Instructions 
PostgreSQL Install Instructions 

 
## Initial Project Setup
Create a new directory to house the project 
```
mkdir djangoproject 
```
Create a virtual environment with: 
```
virtualenv venv 
```
Activate the virtual environment. Once it is activated, you will see the name of the virtual environment in parentheses at the beginning of your command line, like this: (venv)~$ 

```
source venv/bin/activate 
```
### Install Django and Required Packages
```
pip install django 
```
*You may get issues depending on which version of Django you use, if you are running into issues, look at installing a previous version. The latest install worked for me (January ‘23) 

Start a new django project (Don’t forget to add a ‘.’ at the end of the command to ensure proper files are loaded) 

```
django-admin startproject ebdjango .  
```
Now, before doing anything else, we need to set up the local PostgreSQL database for local development. This will be different from the RDS database that we will set up for the live environment, This local database is used to test and develop your Django Web Application before deployment to Elastic Beanstalk. 

Install gunicorn 
```
pip install gunicorn    
```
Install pyscopg2-binary for configuring your local database. (This seems to be the best solution, as I experienced a lot of issues trying to work with the non-binary install of psycopg2.) 
```
pip install psycopg2-binary 
```

Install ebhealthcheck (optional) - If you are running an application load balanced environment, this will ensure that your health checks will work properly. Without it, you will run into health check issues in your environment.  
```
pip install ebhealthcheck 
```
Add ebhealthcheck to settings.py INSTALLED APPS  
```
INSTALLED_APPS = [ 

    ... 

    'ebhealthcheck.apps.EBHealthCheckConfig', 

    ... 

] 
```
Verify that all of your packages are installed with pip freeze 
```
pip freeze 
```
 Elastic Beanstalk will automatically set up the server with all the required packages, but we need to have a file called requirements.txt to communicate what packages to install. Create the requirements.txt file  
```
pip freeze > requirements.txt 
```

## Local PostgreSQL Database Setup   

The first step is to install and set up Postgresql on your machine, check the setup for PostgreSQL for your setup here: PostgreSQL 

1. Create a new database (don’t forget to always have a ‘;’ at the end of each command!). Make sure to remember what name you give your database, as we will need it soon.   
```
CREATE DATABASE databasename;   
```
 2. Create a new user with a new password 
```
CREATE USER user WITH PASSWORD ‘password’;  
```
3. Grant the user privileges on the database 
```
GRANT ALL PRIVILEGES ON DATABASE databasename TO user;  
```
4. Exit with \q 
```
\q  
```

Make sure you have written down your database name, user and password as we need it in the next steps.  

You will then need to go into your main project folder and edit the settings.py file to incorporate the database information that you just setup. 

Under the DATABASES section in your settings.py, edit it to the following making sure to add the information that you used in the earlier step. If you want a copy/paste option, check out the samples contained in this repository.   

```
DATABASES = {  
    'default': {  
        'ENGINE': 'django.db.backends.postgresql_psycopg2',  
        'NAME': 'dbcreatedinpsql',  
        'USER': 'createuser',  
        'PASSWORD': 'createpassword',  
        'HOST': 'localhost',  
        'PORT': '5432',  
    }  
}  

```

### Troubleshooting Postgres Issues 

You may run into some issues with the database not being able to connect. If you are having trouble running the server, such as not being able to connect to the local host, you may need to edit the pg_hba.conf file. This is how you can do that with an Amazon Linux 2 machine:  

Login as the postgres user (you need permission to edit the file)  
```
sudo su postgres  
```
Edit the pg_hba.conf file with a text editor, such as nano 
```
nano /var/lib/psql/data/pg_hba.conf 
```
Edit the file so it looks like the following. You want to change your IPV4 settings to ‘trust’ by replacing the ‘ident’  

(Screen shot here for the pg_hba.conf file)  

Once all of this is done,  
```
sudo systemctl restart postgresql  
```
You can then head back into your project file and activate your virtual environment and migrate your database 
```
python manage.py migrate  
```
## Creating a Superuser for the Database 

Once you have set up your local database, you can create a superuser in order to administrate your application. This will be your local superuser, when we set up the RDS for the live environment, we will set up the live superuser as well as automate that process.  

1. Create superuser 
```
python manage.py createsuperuser  
```
 Run your Django site locally 

Run the Django site locally to see if the install worked:  
```
python manage.py runserver 
```
Check the site in a browser at 127.0.0.1:8000  

If you made it here, you have successfully installed django and the basic requirements for deploying it to AWS Elastic Beanstalk!
 

## Configuring the project for Elastic Beanstalk 

Now that you have set up your project on the local server, you can set up your configuration for deployment to AWS Elastic Beanstalk. Make sure you have your AWS credentials set up and configured as well as the awsebcli installed and set up.  

Make sure the virtual environment is activated 

Now, we need to specify more settings for the Elastic Beanstalk Environment. For that, we need to create a directory and call it: .ebextensions: 
```
mkdir .ebextensions 
```
Now, you want to create a file within that .ebextensions directory and call it django.config. (See sample files for an example) 

Open that file in a text editor and add the following text. Be mindful of your spacing, check the sample documents contained in this repo if you want a copy/paste option that has proper spacing:  
```
option_settings:	 
  aws:elasticbeanstalk:container:python: 
    WSGIPath: ebdjango.wsgi:application 
```

Add a Procfile – Create a file called “Procfile.txt” and add the following, making sure to substitute the name of your project  
```
web:gunicorn --bind:8000 --workers 3 --threads 2 <your_project_name>.wsgi:application 
```
Next, we are going to deploy our application to Elastic Beanstalk. There is still more to go after this step though, as we are going to have to set up a live RDS Postgres Database so that our application will be able to work. 

 
## Deploy the Application using the EB CLI  

Initialize the repository with eb init command: 
```
eb init 
```
Answer the question prompts to make sure you are setting it up in the correct region and with an SSH key (optional) as well as specifying the type of load balancer. For this project, selet Application Load Balancer. 

Create an environment and deploy the application to it with eb create. This will ask you about the name, etc. 
```
eb create 
```

Once you have completed that, it will go through and create the resources (may take a couple of minutes) 

Use eb status after everything is created to check the status.
```
eb status 
```
Copy the CNAME property (this is the domain name of where the application is located)  

Open the settings.py file and locate the ALLOWED_HOSTS setting and add the CNAME into that section.  

Save that file and run eb deploy to update that: eb deploy 



## Set up a and configure a new PostgreSQL RDS database 

Verify that you have imported os in your settings.py file. 

```
import os
```

First, go to the elastic beanstalk page in the console 

Click on configuration link 

Create a new RDS database 

Change the DB engine to postgresql 

Add a username and password 

Save the changes 

This will take a while, so in the meantime, head into the settings.py and update the DATABASE section to include the settings for utilizing the RDS 


Then, you need to update your settings.py to look like the following: 

```
if 'RDS_DB_NAME' in os.environ:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': os.environ['RDS_DB_NAME'],
            'USER': os.environ['RDS_USERNAME'],
            'PASSWORD': os.environ['RDS_PASSWORD'],
            'HOST': os.environ['RDS_HOSTNAME'],
            'PORT': os.environ['RDS_PORT'],
        }
    }


else:
    DATABASES = {
    'default': {  
        'ENGINE': 'django.db.backends.postgresql_psycopg2',  
        'NAME': 'dbcreatedinpsql',  
        'USER': 'createuser',  
        'PASSWORD': 'createpassword',  
        'HOST': 'localhost',  
        'PORT': '5432',  
    }  
}  

```
You can find this info at realpython.com/deploying-a-django-app-to-aws-elastic-beanstalk/#configuring-a-database , which was very helpful to me in getting this set up!



## Automating Live Database migrations:  

Once you have your live site set up, you don’t want to have to login and do all of the migrations manually, and so for that we can add another configuration file which will tell elastic beanstalk to make the necessary changes.  

Add a configuation file in .ebextensions called eb-commands.config (see instructions here https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create-deploy-python-django.html  



Deploy with eb deploy after everything is updated 


## Automate the createsuperuser command for your live site 

The easiest (and simplest) way to create a superuser for your live database is to put it into your configuration file.  

Create a new app with django admin 

Create a new directory within it called management 

Create a new file called createsuper.py 

Add the following code to the createsuper.py file 

Add the following code to your configuration file:  

 

Now, you will have a superuser created when you deploy your site. No need to get rid of the code, unless you want to be really strict with security, as the code will only create the superuser if there is not already a superuser with the account name.  

 

## Collecting Static Files and Automating the process 

Tell Django where to Collect the static files by updating the settings.py with the following:      

  

  Screen shot 

 

 Static files (CSS, JavaScript, Images)   

 https://docs.djangoproject.com/en/2.2/howto/static-files/     

  

STATIC_URL = '/static/'   

STATIC_ROOT = 'static'     

  

  

Collect the static files with      

~python manage.py collectstatic   

  

    

### Optional (but vital to your sanity!) 

Set up your version control 

It’s vital to make sure you have some sort of version control set up, whether it is through AWS CodeCommit or Github but that is beyond the scope of this tutorial. Consider this a friendly reminder 😊  

Bonus: 

Create a .gitignore file for your django project. There is a great starter template for your django .gitignore file, which is really helpful for setting up git with django. Link to .gitignore file. 

 

 

 

 

______ 

 

 

 

 

 

 

 

 