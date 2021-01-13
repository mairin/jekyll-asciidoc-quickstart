# Client-side workflow

## Working with the Chris REST API

The Chris REST API uses the standard [Collection+JSON](http://amundsen.com/media-types/collection/) hypermedia type to exchange resource representations with clients. All the functionality provided by the API can be discovered by clients by doing GET requests to links ("href" elements) presented by the hypermedia documents returned by the web server, starting with the API's "home page". The "home page" relative url would be: <tt>/api/v1/</tt>

In order to run a plugin app with specific arguments a client agent can follow the following workflow:

````
1. GET /api/v1/ 
2. GET url with link relation "plugins" in the previous response. Eg.: GET /api/v1/plugins/
3. Look for the desired plugin in the items collection of the previous response and then GET url with link relation "instances" in the previous response. Eg.: GET /api/v1/plugins/1/instances/
4. Fill out the template object of the previous response and then do a POST request to the same url. Eg.: POST /api/v1/plugins/1/instances/ template:={data...}
````

There are two types of plugin apps: "fs" and "ds". The former is given to plugin apps that when run will create a new data feed. They don't require a value for the "previous" property of the template object as they are the first app run in a pipeline of plugin apps. In opposition plugin apps of type "ds" require the previous application of another plugin app and act on the outputs of that app. They do require the id of the previously run plugin app as the value for the "previous" property of the template object. 

## Example

Let's assume you are using default Django's development server and [httpie](https://github.com/jkbrzt/httpie) client. The following "conversation" shows how to interact with a workflow consisting of a "fs" and "ds" plugin:

* First create a system user to be able to do authenticated requests. We are going to create user "bob" with password "bob-pass":
````
python manage.py createsuperuser
````

* Start the Django development server:
````
python manage.py runserver
````

* In another terminal Get the "home page" url:
````
http -a bob:bob-pass http://127.0.0.1:8000/api/v1/

HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/vnd.collection+json
Date: Tue, 09 Aug 2016 20:22:04 GMT
Server: WSGIServer/0.2 CPython/3.5.1+
Vary: Accept, Cookie
X-Frame-Options: SAMEORIGIN

{
    "collection": {
        "href": "http://127.0.0.1:8000/api/v1/",
        "items": [],
        "links": [
            {
                "href": "http://127.0.0.1:8000/api/v1/plugins/",
                "rel": "plugins"
            }
        ],
        "version": "1.0"
    }
}
````

* Get the plugin list using the link relation "plugins" in the previous response:
````
http -a bob:bob-pass http://127.0.0.1:8000/api/v1/plugins/

HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/vnd.collection+json
Date: Tue, 09 Aug 2016 20:23:18 GMT
Server: WSGIServer/0.2 CPython/3.5.1+
Vary: Accept, Cookie
X-Frame-Options: SAMEORIGIN

{
    "collection": {
        "href": "http://127.0.0.1:8000/api/v1/plugins/",
        "items": [
            {
                "data": [
                    {
                        "name": "name",
                        "value": "simpledsapp"
                    },
                    {
                        "name": "type",
                        "value": "ds"
                    }
                ],
                "href": "http://127.0.0.1:8000/api/v1/plugins/7/",
                "links": [
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/7/parameters/",
                        "rel": "parameters"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/7/instances/",
                        "rel": "instances"
                    }
                ]
            },
            {
                "data": [
                    {
                        "name": "name",
                        "value": "simplefsapp"
                    },
                    {
                        "name": "type",
                        "value": "fs"
                    }
                ],
                "href": "http://127.0.0.1:8000/api/v1/plugins/6/",
                "links": [
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/6/parameters/",
                        "rel": "parameters"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/6/instances/",
                        "rel": "instances"
                    }
                ]
            }
        ],
        "links": [
            {
                "href": "http://127.0.0.1:8000/api/v1/",
                "rel": "feeds"
            }
        ],
        "version": "1.0"
    }
}
````

* Get the specific 'fs' plugin's instances list using the link relation "instances" in the previous response:
````
http -a bob:bob-pass http://127.0.0.1:8000/api/v1/plugins/6/instances/

HTTP/1.0 200 OK
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/vnd.collection+json
Date: Tue, 09 Aug 2016 20:27:29 GMT
Server: WSGIServer/0.2 CPython/3.5.1+
Vary: Accept, Cookie
X-Frame-Options: SAMEORIGIN

{
    "collection": {
        "href": "http://127.0.0.1:8000/api/v1/plugins/6/instances/",
        "items": [],
        "links": [
            {
                "href": "http://127.0.0.1:8000/api/v1/plugins/6/",
                "rel": "plugin"
            }
        ],
        "template": {
            "data": [
                {
                    "name": "previous",
                    "value": ""
                },
                {
                    "name": "dir",
                    "value": ""
                }
            ]
        },
        "version": "1.0"
    }
}
````

* Perform a POST request to the previous url with a data payload consisting of the template in the previous response filled out (plugins of type "fs" will ignore a value for the "previous" property if provided):
````
http -a bob:bob-pass POST http://127.0.0.1:8000/api/v1/plugins/6/instances/ Content-Type:application/vnd.collection+json Accept:application/vnd.collection+json template:='{"data":[{"name":"dir","value":"./"}, {"name":"previous","value":""}]}'

HTTP/1.0 201 Created
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/vnd.collection+json
Date: Tue, 09 Aug 2016 20:33:09 GMT
Location: http://127.0.0.1:8000/api/v1/plugins/instances/37/
Server: WSGIServer/0.2 CPython/3.5.1+
Vary: Accept, Cookie
X-Frame-Options: SAMEORIGIN

{
    "collection": {
        "href": "http://127.0.0.1:8000/api/v1/plugins/6/instances/",
        "items": [
            {
                "data": [
                    {
                        "name": "id",
                        "value": 37
                    },
                    {
                        "name": "plugin_name",
                        "value": "simplefsapp"
                    },
                    {
                        "name": "owner",
                        "value": "bob"
                    }
                ],
                "href": "http://127.0.0.1:8000/api/v1/plugins/instances/37/",
                "links": [
                    {
                        "href": "http://127.0.0.1:8000/api/v1/20/",
                        "rel": "feed"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/6/",
                        "rel": "plugin"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/string-parameter/36/",
                        "rel": "string_param"
                    }
                ]
            }
        ],
        "links": [],
        "version": "1.0"
    }
}
````

* Now the id (37) in the previous response can be used to fill out the value of the property "previous" of a template that could be used in a POST request to the list of instances of a "ds" plugin:
````
http -a bob:bob-pass POST http://127.0.0.1:8000/api/v1/plugins/7/instances/ Content-Type:application/vnd.collection+json Accept:application/vnd.collection+json template:='{"data":[{"name":"prefix","value":"myprefix"}, {"name":"previous","value":"37"}]}' 

HTTP/1.0 201 Created
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/vnd.collection+json
Date: Wed, 10 Aug 2016 18:23:40 GMT
Location: http://127.0.0.1:8000/api/v1/plugins/instances/41/
Server: WSGIServer/0.2 CPython/3.5.1+
Vary: Accept, Cookie
X-Frame-Options: SAMEORIGIN

{
    "collection": {
        "href": "http://127.0.0.1:8000/api/v1/plugins/7/instances/",
        "items": [
            {
                "data": [
                    {
                        "name": "id",
                        "value": 38
                    },
                    {
                        "name": "plugin_name",
                        "value": "simpledsapp"
                    },
                    {
                        "name": "owner",
                        "value": "jbernal"
                    }
                ],
                "href": "http://127.0.0.1:8000/api/v1/plugins/instances/38/",
                "links": [
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/instances/37/",
                        "rel": "previous"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/7/",
                        "rel": "plugin"
                    },
                    {
                        "href": "http://127.0.0.1:8000/api/v1/plugins/string-parameter/40/",
                        "rel": "string_param"
                    }
                ]
            }
        ],
        "links": [],
        "version": "1.0"
    }
}
````


# Server-side workflow

## Developing a Chris plugin app

Chris plugin apps are Python packages that contain a module with the same name as the package. The module can be run from the command-line with a series of previously defined arguments to perform any predefined computing task. 

The Python module must define a single class that inherits from <tt>ChrisApp</tt>, the base class for all Chris plugin apps. Two methods are required to be overridden:

* <tt>define_parameters(self)</tt>
* <tt>run(self, options)</tt>

The first method can be overridden to define the arguments expected by the Chris plugin app. This can be done by calling the <tt>add_argument</tt> method to individually define each single argument. This method accepts the same arguments as the [add_argument](https://docs.python.org/3/library/argparse.html#the-add-argument-method) method of the <tt>ArgumentParser</tt> class defined in the Python's standard [argparse](https://docs.python.org/3/library/argparse.html#module-argparse) module.

The second method can be overriden to define any computing task. The options argument is an object containing the arguments passed to the plugin app as attributes plus two additional attributes:

* <tt>options.inputdir</tt> , directory containing the input data files (only available for plugin apps of type 'ds')
* <tt>options.outputdir</tt> , directory where the app's output data files must be written



## Adding/removing a Chris plugin app 

A new Chris plugin app can be added to the system by an administrator by first placing the corresponding python package in the appropriate subdirectory as explained above. The administrator can then register the plugin app with the system by running the plugin app manager which is a Python command-line utility also located under the <tt>services</tt> package in the <tt>plugins</tt> django app. Eg.:

````
python manager.py --add simplefsapp
````
After this the REST system will be aware of the new plugin app and will be able to respond to related HTTP requests.

To remove the plugin app the administrator can run:

````
python manager.py --remove simplefsapp
````
