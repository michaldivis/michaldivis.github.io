
# How to install PostgreSQL 12.1 from binaries

*NOTE: This tutorial talks about PostgreSQL version 12.1 but should be applicable to pretty much any version.*

Recently, I needed to install the PostgreSQL database along with my WPF app. However, putting the PostgreSQL 12.1 installer into my app's installer as a custom action just didn't work for some reason. The PostgreSQL installer kept shutting down and I didn't want to spare anymore precious time on a solution I didn't even really like.

So I've done some digging and it turned out, PostgreSQL has been kind enough to provide the raw bindary files, so you can install it your self. This process has been documented in a few blogposts I've found, however, none of them showed how to make your database run on a different that the default port. And that is something I wanted. Since I'm installing the database specifically for my app, I want the service name to be something like "myAppPostgreSQL12" and run it on a different port, so I don't collide with possible preinstalled versions of PostgreSQL.

Let's do this!

## 1 - Download the binaries
Download the binaries zip file from [here](https://www.postgresql.org/download/)
## 2 - Extract the binaries

Extract the files and put the contents of the pgsql folder wherever you like, for this example, we'll put them in C:\Users\User\Desktop\CustomPostgreSQL
## 3 - Create the data folder

Create a folder called data in the C:\Users\User\Desktop\CustomPostgreSQL
## 4 - Launch CMD and cd into the bin directory

Now launch CMD as administrator and use this command to go to the bin directory
cd C:\Users\User\Desktop\CustomPostgreSQL\bin
## 5 - Create a database cluster

Now let's create a db cluster. In this step, there are two options of setting the superuser password: manually setting the password from console, reading the password from a file.

Manually setting the password from console:

`initdb -U postgres -A password -E utf8 -W -D "C:\Users\User\Desktop\CustomPostgreSQL\data"`

Reading the password from a file (* it reads whatever is on the first line of the file and uses that as a password. The file pw.txt is created by me, it can by named whatever and be wherever you like):

`initdb -U postgres -A password -E utf8 --pwfile "C:\Users\User\Desktop\CustomPostgreSQL\pw.txt" -D "C:\Users\User\Desktop\CustomPostgreSQL\data"`

There are many parameters this can take, so I'll go over the ones used here:

* `-U postgres` is the name of the superuser
* `-A password` is an authentication method
* `-E utf8` is the default encoding
* `-W` means the program will prompt you to create a password manually
* `-D` is a path to the data directory
* `--pwfile` is a path to the file that contains the password

You can take a look at all the parameters here
## 6 - Register the database as a service

Finally, we register the thing as a windows service and we're done.

`pg_ctl register -N "OurCustomPostgreSQLService" -U "LocalSystem" -D "C:\Users\User\Desktop\CustomPostgreSQL\data" -w -o "-p 5434"`

Let's go over the parameters again:

* `-N "OurCustomPostgreSQLService"` sets the name of the service to OurCustomPostgreSQLService
* `-U "LocalSystem"` sets the user that's going to be running the service to LocalSystem user
* `-D "C:\Users\User\Desktop\CustomPostgreSQL\data"` sets the path to our data directory
* `-w` just tells the program to wait for the result and report success/failure
* `-o "-p 5434"` sets to port the service is gonna be running on to 5434 instead of the default 5432

## 7 - Add some service description (optional)

If you want, you can set a description for the service so anyone else might be able to tell what it is.

`sc description OurCustomPostgreSQLService "A custom PostgreSQL service running on port 5434"`

## Wrapping up

I'll update this post if I find out any other useful information. Cheers!
