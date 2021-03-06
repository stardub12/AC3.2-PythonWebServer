# A Web Server in Python

* Hosting: AWS EC2
* Webserver: Apache
* Language: Python
* Framework: Flask

## EC2 - Elastic Cloud Computing

### Creating

1. Create an EC2 instance 
1. New security group
1. New keypair

### Connecting

> Locally (on your laptop)

After you download the key (pem file) from AWS, save it to the .ssh directory in your 
home directory.

```sh
  mv got.pem ~/.ssh
```

Change the permissions on that file in order for ssh to work.

```sh
  chmod 400 got.pem
```

Access the remote computer. ssh = Secure shell.

```sh
ssh -i ~/.ssh/got.pem ec2-user@ec2-54-164-88-224.compute-1.amazonaws.com
```

> This video is pretty good:
> https://www.youtube.com/watch?v=M2Wc8JIS-p8
> However:
> I’d recommend keeping all your keys in ~/.ssh/


## Installations

### Environment
```sh
$ sudo yum update
```

### Apache (Web Server)
```sh
$ sudo yum install -y httpd24 
```
### Python/Apache interoperability
```sh
$ sudo yum install mod24_wsgi-python27.x86_64
```

### Flask framework
```sh
$ sudo pip install flask
```

## Operation

Run the web server

```sh
$ sudo service httpd restart
```

## Flask

Reference 
* http://flask.pocoo.org/docs/0.12/quickstart/

Flask in AWS (one way):
* http://www.datasciencebytes.com/bytes/2015/02/24/running-a-flask-app-on-aws-ec2/

Some differences in our environment using AWS's default server. For one thing,
symbolically linking into the ec2-user directory required setting permissions
on that user's directory that prevented subsequent ssh sessions. Therefore, do not
run ```sudo ln -sT ~/flaskapp /var/www/html/flaskapp``` or change the permissions
on the home directory.

```sh
# this is the root directory of the server
$ cd /var/www/html

# create our directory as root
$ sudo mkdir flaskapp

# root can make us the owner again
$ sudo chown ec2-user flaskapp/

$ cd flaskapp

$ echo "Hello World" > index.html
```

### Test access to flaskapp directory

At this point the url http://<your server>/flaskapp/ should display "Hello World". The following 
steps set up the flask framework to handle requests and routes.

## Configure Apache to see Flask

Edit /etc/httpd/conf.modules.d/10-wsgi.conf (or /etc/httpd/conf/httpd.conf 
if the former doesn't exist):

```
$ sudo nano /etc/httpd/conf.modules.d/10-wsgi.conf
```

Adding these lines:

```
WSGIDaemonProcess flaskapp threads=5
WSGIScriptAlias / /var/www/html/flaskapp/flaskapp.wsgi

<Directory flaskapp>
    WSGIProcessGroup flaskapp
    WSGIApplicationGroup %{GLOBAL}
    Order deny,allow
    Allow from all
</Directory>
```

Restart Apache

```sh
$ sudo service httpd restart
```

### Trivial Flask App

Save this configuration to a file named flaskapp.wsgi. You can use ```cat``` and redirection ```>``` to do this:

```sh
cat > flaskapp.wsgi
```

This will put the cursor on the next line. Paste in the content you want, make sure 
the cursor is on the beginning of a line and type CTRL-D. You will have a file named 
flaskapp.wsgi with the contents you typed.

```
import sys
sys.path.insert(0, '/var/www/html/flaskapp')

from flaskapp import app as application
```

Do the same thing for the python/Flask script.

```sh
cat > flaskapp.py
```

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
  return 'Hello from Flask!'
  
if __name__ == '__main__':
  app.run()

```

## Database / Python
   
### Installation

```sh
$ sudo yum install python27-pip
$ sudo yum install python27-devel
$ sudo yum install mysql27-devel
$ sudo yum install MySQL-python27
```

## Flask Code with Database

```python
from flask import Flask
from flask import json
from flask import jsonify

import MySQLdb

app = Flask(__name__)

@app.route('/')
def hello_world():
  return 'Hello from Flask!'

@app.route('/api/characters')
def characters():
  db = dbConnection()

  ret = ''
  # you must create a Cursor object. It will let
  #  you execute all the queries you need
  cur = db.cursor()

  # Use all the SQL you like
  cur.execute("""select c.name, h.name as house
  from houses as h, characters as c, characters_houses as ch
  where ch.house_id = h.id
  and ch.character_id = c.id""")

  # gather all the results in this array
  characters = []
  for row in cur.fetchall():
      name = row[0]
      house = row[1]

      # create an ad-hoc object
      characters.append({"name":name, "house":house})

  db.close()

  return jsonify(characters)

def dbConnection():
  conn = MySQLdb.connect(host="atlas.cf626xxbuyrf.us-east-1.rds.amazonaws.com",    # your host, usually localhost
                     user="gotuser",         # your username
                     passwd="winteriscoming",  # your password
                     db="got")        # name of the database
  return conn

if __name__ == '__main__':
  app.run()
```

## Monitoring

```sh
# hits
$ sudo tail -f /var/log/httpd/access_log

# errors
$ sudo tail -f /var/log/httpd/error_log
```

## Troubleshooting

* Check the error log for hints about errors
* Check the access log if you suspect the request isn't even being processed
* Be sure you have firewall access (i.e. port 80 is open in Security Groups)
* If new code seems not to be running or if all else fails try restarting apache
  ```sudo service httpd restart```
  

## Other References

1. Helpful but in PHP
    http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateWebServer.html
