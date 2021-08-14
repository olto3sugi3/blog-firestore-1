# Use multiple Firebase projects properly in WEB application development
I like Firebase Firestore and Authentication because they are very easy to use and are great for creating web apps.
In the process of using Firebase in various cases, I needed to use Firebase projects properly in development and production environments and copy data between different Firebase projects, so I thought about what to do.
The source code is made in Python, but I think that it can be handled with the same idea in other languages, so please refer to it.
## How to access Firebase
We prohibit direct access to the Firestore from a browser - Javascript -. Since it is a WEB application that is open to the public, I was worried about attacks from the browser, so the rule of Firestore is set to [[Locked mode]](https://firebase.google.com/docs/firestore/quickstart?authuser=0#locked-mode).

```js
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```
Since it is not accessed from the client, you will be using the [[Firebase Admin SDK]](https://firebase.google.com/docs/admin/setup) in Python on the server.
Accessing Firestore with administrator privileges uses the key file of the service account, and this time we will use multiple key files properly.
## Advance preparation
### 1.Prepare the key file of the service account
Since we are using multiple Firebase projects, we will generate private keys of service accounts for each project and save the JSON file.
A new private key can be created on the Firebase console with Project Overview => Project Settings => Service Account Key.
Create an arbitrary folder on the server with the file name [Firebase project ID] .json and save it. I made /usr/firebase/serviceAccountKey to save files.
###2.Set the folder name in the environment variable
Pass this folder name to CGI with the environment variable. My server runs CGI with nginx + fcgiwrap, so set it at the bottom of /etc/nginx/fcgiwrap.conf as follows.

```fcgiwrap.conf
location /cgi-bin/ {
    ・・・・・・
    fastcgi_param FIREBASE_CREDENTIALS [the Folder name you have made];
}
```
The setting method here differs depending on your system - such as apache - , so please set it according to your own server.
Now you are ready to go.

## Python program that returns Firebase (app) when you specify a Firebase project ID
I made it with Python.

```py
import os
import json
import firebase_admin
from firebase_admin import credentials, auth, firestore

def initapp(my_project):
    # get Firebase (app)
    # from list of already initialized Firebase(app) find correct one
    for myapp in firebase_admin._apps:
        if firebase_admin.get_app(myapp).project_id == my_project:
            return firebase_admin.get_app(myapp)
    # Create new Firebase(app), If not found
    try:
        mycert = os.environ["FIREBASE_CREDENTIALS"] + my_project + '.json'
        cred = firebase_admin.credentials.Certificate(mycert)
        app = firebase_admin.initialize_app(cred, name=my_project) # Be sure to specify name
        return app
    except:
        if "[DEFAULT]" in firebase_admin._apps:
            return firebase_admin.get_app()
        else:
            # Use the application default credentials = GOOGLE_APPLICATION_CREDENTIALS 
            cred = credentials.ApplicationDefault()
            default_app = firebase_admin.initialize_app(cred)
            return default_app

```
When calling this, pass the Firebase project ID as an argument.
If the project's Firebase has already been initialized, it returns Firebase (app). If it hasn't been initialized, it will initialize and then return Firebase (app). If project ID is not found, it initializes and returns the default Firebase.
When using multiple Firebases properly like this, you need to specify `firebase_admin.initialize_app (cred, name = my_project)` and name. This allows you to identify multiple Firebase Projects. I'm using this name when searching for an initialized Firebase.

## How to use this
### 1.Use multiple Firebase Projects at the same program
If you want to copy data between different Firebases, do the following:

```py
# copy collection
from_project = [Project ID of Source Firebase]
from_app = initapp(from_project)    # initialize Firebase
fromdb = firestore.client(from_app) # get Firestore from initialized Firebase
to_project = [Project ID of Destination Firebase]
to_app = initapp(to_project)        # initialize Firebase
todb = firestore.client(to_app)     # get Firestore from initialized Firebase

# cpoy data from "fromdb" to "todb"
from_ref = fromdb.collection([Source collection]).document([Source document-ID])
mydoc = from_ref.get()
myrec = mydoc.to_dict()
target_ref = todb.collection([Destination collection]).document([Destination document-ID])
target_ref.set(myrec)
......

```

### 2.Distinguish between development and production Firebase
I made such a matrix with json and saved it on the server. Then, the save destination of this matrix is also passed to CGI as an environment variable.

```json
{
    "hello-world":{
          "www.[domain name]":"[Production Firebase Project ID #1]"
        , "test.[domain name]":"[Test Firebase Project ID #2]"
        , "test.local.[domain name]":"[Test Firebase Project ID #2]"
    }
    , "[WEBアプリの識別名]":{
          "www.[domain name]":"[Production Firebase Project ID #3]"
        , "test.[domain name]":"[Test Firebase Project ID #4]"
        , "test.local.[domain name]":"[Test Firebase Project ID #4]"
    }
    ......

}
```
Use this matrix to identify Firebase ID and connect to desired Firebase.
There are multiple web apps running on my server, but each web app has its own "application identifier". I save this "application identifier" in the resource file.

```json
{
    "app_id":[application identifier]
    ......
}
```
This app_id is passed as json data when calling CGI from JavaScript (POST).
The CGI program identifies the Firebase project ID from the matrix, the "application identifier" and the web server name, and connect to correct Firebase.

```py
# identifie FIrebase Project ID from "application identifier" and "web server name"
yapp_id = postJson['app_id'] # application identifier that is passed from JavaScript
my_host = os.environ['SERVER_NAME'] # web server name the CGI program is running
with open(os.environ['environment variable that shows destination of the matrix']) as firebaseProjects:
    projectJson = json.load(firebaseProjects)
    my_project = projectJson[myapp_id][my_host]

app = initapp(my_project) # initialize Firebase
db = firestore.client(app) # get Firestore from initialized Firebase

# Execute processing using Firestore
newdocument = db.collection("[collection]").document()
......
```
**<font color="Blue">* Advantages of this method</font>**
This setting is not necessary if it is troublesome. forget it.
I like the fact that the exact same program runs in different environments on the production server, the development server, and VS Code for coding.
The advantage over that is that you can use the same program in another web application. Taking advantage of this characteristic, programs that can be used in common by multiple WEB applications are made into parts and deployed as common programs.
When you call CGI, you can absorb the difference in the environment just by changing the "application identifier".

#### appendix
The reason why the save folder name of the key file of the service account and the destination of the matrix are specified by environment variables without hard coding is that I wanted to make exactly the same Python work on the server (Linux) and the coding PC (Windows). Of course, there is no production key file on the PC.
The "test.local. [Domain name]" defined in the matrix is the server name set in Go Live of VS Code on the coding PC. CGI cannot be called from the VSCode WEB server, but "test.local. [Domain name]" is set in the environment variable "SERVER_NAME" on each PC. When debugging Python with VSCode, follow the matrix and go to connect to Firebase of test environment.
Furthermore, if you want to separate the environment for coding (UT) and test server (IT), you can easily separate them because you only need to define them in the matrix.

### Appendix
I searched on the WEB, but I was at a loss because I couldn't find a site that explains all of them. I've made something very useful, so I'll publish it here.

## Plese visit our website.
### [WEB System Infrastructure Guide for Beginners](https://www.olto3-sugi3.tk/index.html)

