# Debugging CUBE pacsquery  and  pacsretrieve without containers

## Abstract

This page documents the control of the containerized pacsquery and pacretrieve plugins outside of the whole containerized CUBE. In other words, all services are instantiated "on the metal" to allow for better flow understanding and debugging.

## Introduction
Due to the distributed and containerized nature of CUBE, debugging the ancillary services, ``pfcon``, ``pfioh``, and ``pman`` can be particularly difficult. The most effective method of debugging is to run the services non-containerized, but preserving the pattern of volume mounts using symbolic links in the host filesystem, followed by a typical CUBE directive.

## Preconditions

### Volume linking
Create the following paths on the host FS -- in any directory. I typically do this in the ``ChRIS_ultron_backend`` dir, but for debugging the services outside of ChRIS, any directory (like ``~/tmp``) will be fine:

```bash
mkdir -p ./FS/remote # Simulate remote FS
chmod 777 ./FS/remote

# To debug "local" services, uncomment the following
# (Note that at first this is probably not necessary)
# mkdir -p ./FS/local  # Simulate local FS
# chmod 777 ./FS/local

sudo mkdir /hostFS
sudo ln -s $(pwd)/FS/remote /hostFS/storeBase    # Simulates the FS within the pman/pfioh 
                                                 # containers

# If you are trying to debug *all* services, including 'local' ones, 
# uncomment the below
#sudo ln -s $(pwd)/FS/local  /hostFS/pfconFS     # Simulates the FS within the pfcon/cube 
                                                 # containers

sudo mkdir -p /usr/users                         # Explicitly maps the CUBE internal path
sudo chmod 777 /usr/users
```

The ``storeBase`` will be used by ``pfioh`` and ``pman`` and simulates the remote FS. The ``pfconFS`` and ``/usr/users`` simulates the local and CUBE DB trees.

### HOST_IP

You *must* export an environment variable, <tt>HOST_IP</tt> to the terminal running ``pfurl`` and ``pfcon``. On Linux this is:

```bash
export HOST_IP=$(ip route | grep -v docker | awk '{if(NF==11) print $9}')
```

### Gotchas!

A source of some frustration can be the ``pfurl`` module which is used internally by ``pfcon``. If debugging ``pfurl`` code while debugging the interaction of the services, it is a good idea to link the ``pfurl.py`` module in the virtualenv to the file being edited. In my setup, this is effected by:

```bash
cd ~/src/python-env/chris_env/lib/python3.5/site-packages/pfurl
mv pfurl.py pfurl.orig.py
ln -s ~/src/pfurl/pfurl/pfurl.py .
```

Alternatively, if you make changes to ``pfurl`` and propagate these all the way through the git repo, docker hub, and up to PiPy, then you can install this laters ``pfurl`` in the virtualenv for ``pfcon`` to use:

```bash
pip3 install pfurl==X.Y.Z
```
where ``X.Y.Z`` is the version of ``pfurl`` to install.

## Debug setup

### Create an "environment"

The simplest way to debug is to open four terminals. Each terminal will be used to run a specific service. Typically from left to right the terminals should be anchored in the following source repositories:


| Terminal 1    | Terminal 2    | Terminal 3  | Terminal 4 |
| ------------- |:-------------:| -----------:| ----------:| 
| pfurl         | pfcon         | pfioh       | pman       |

### Run the services

In each terminal, ``cd`` to the ``bin`` dir of each, and run the respective service from terminal

```bash
./pfcon --forever --httpResponse
./pfioh --forever --httpResponse --createDirsAsNeeded --storeBase /hostFS/storeBase
rm -fr /tmp/pman ; ./pman --rawmode 1 --http --port 5010 --listeners 12
```
### Workflow

It is important that all the services need to stop and be restarted each time a change is made. In particular, the ``pman`` service needs its database cleared if the same job is being submitted repeatedly during debugging.

**If you don't <tt>rm -fr /tmp/pman</tt> before each call to ``pman`` you WILL get unpredictable results and behaviour!**

### Swarm issues and re-scheduling the same service 

In debugging one often will repeat the same ``pfurl`` command and associated JSON over and over again. Within this JSON is a directive specifying the service name to the swarm manager. Sometimes, the service remains in the swarm scheduler and if the same ``pfurl`` is resent with the same service name, ``pman`` will throw an internal exception that is hard to notice.

The solution is to check on the swarm service using

```bash
dsl
```

and if it exists, to remove the service

```bash
dss <service>
```

where ``dsl`` (docker service list) is a shell alias

```bash
alias dsl="docker service ls "
```

and ``dss`` (docker service stop) another alias

```bash
alias dss="docker service rm "
```

## Simulate CUBE calls

### ``hello``

Assuming satisfied preconditions, let's say ``hello`` to ``pfcon``. It will in turn ask each of ``pfioh`` and ``pman`` ``hello`` and return the response.

```bash
./pfurl --verb POST --raw --http ${HOST_IP}:5005/api/v1/cmd \
        --httpResponseBodyParse \
        --jsonwrapper 'payload' --msg \
'{  "action": "hello",
    "meta": {
                "askAbout":     "sysinfo",
                "echoBack":      "Hi there!",
                "service":       "host"
            }
}'
```
### Status

To find the status on a job, say job ``2`` as in the ``pl-pacsretrieve`` examples below, use

```bash
./pfurl --verb POST --raw --http ${HOST_IP}:5005/api/v1/cmd \
        --httpResponseBodyParse --jsonwrapper 'payload' \
        --msg \
'{  "action": "status",
    "meta": {
                "remote": {
                               "key":          "2"
                          },
                "service":       "host"
            }
}'
```


### Call a pl-pacsqery FS plugin

In these calls, be sure that the ``HOST_IP`` env variable is set correctly, also make sure that the ``pfdcm`` server is in fact running and so too the ``orthanc`` server!

First, a ``pl-pacsquerry`` plugin:

```bash
./pfurl --verb POST --raw \
        --http ${HOST_IP}:5005/api/v1/cmd \
        --httpResponseBodyParse --jsonwrapper 'payload' \
        --msg '
        {   "action": "coordinate",
            "threadAction":     true,
            "meta-store": {
                        "meta":         "meta-compute",
                        "key":          "jid"
            },

            "meta-data": {
                "remote": {
                        "key":          "%meta-store"
                },
                "localSource": {
                        "path":         "/data/dicomDir"
                },
                "localTarget": {
                        "path":         "/hostFS/pfconFS/chris/feed_1/data",
                        "createDir":    true
                },
                "specialHandling": {
                        "op":           "plugin",
                        "cleanup":      true
                },
                "transport": {
                    "mechanism":    "compress",
                    "compress": {
                        "encoding": "none",
                        "archive":  "zip",
                        "unpack":   true,
                        "cleanup":  true
                    }
                },
                "service":              "host"
            },

            "meta-compute":  {
                "cmd":      "$execshell $selfpath/$selfexec /share/outgoing --saveinputmeta --pfdcm 10.17.24.163:5015 --PatientID LILLA-9731 --PACSservice orthanc --pfurlQuiet --summaryKeys PatientID,PatientAge --summaryFile summary.txt --resultFile results.json --numberOfHitsFile hits.txt",
                "auid":     "rudolphpienaar",
                "jid":      "1",
                "threaded": true,
                "container":   {
                        "target": {
                            "image":            "fnndsc/pl-pacsquery",
                            "cmdParse":         true
                        },
                        "manager": {
                            "image":            "fnndsc/swarm",
                            "app":              "swarm.py",
                            "env":  {
                                "meta-store":   "key",
                                "serviceType":  "docker",
                                "shareDir":     "%shareDir",
                                "serviceName":  "1"
                            }
                        }
                },
                "service":              "host"
            }
}'       
```

### Now call the pl-pacsretrieve

In this call, be sure that the ``HOST_IP`` env variable is set correctly. Now, a call to the ``pl-pacsretrieve``:

```bash
pfurl --verb POST --raw \
      --http ${HOST_IP}:5005/api/v1/cmd \
      --httpResponseBodyParse --jsonwrapper 'payload' \
      --msg '
        {   "action": "coordinate",
            "threadAction":     true,
            "meta-store": {
                        "meta":         "meta-compute",
                        "key":          "jid"
            },

            "meta-data": {
                "remote": {
                        "key":          "%meta-store"
                },
                "localSource": {
                        "path":         "/hostFS/pfconFS/chris/feed_1/data"
                },
                "localTarget": {
                        "path":         "/hostFS/pfconFS/chris/feed_1/pl-pacsretrieve/data",
                        "createDir":    true
                },
                "specialHandling": {
                        "op":           "plugin",
                        "cleanup":      true
                },
                "transport": {
                    "mechanism":    "compress",
                    "compress": {
                        "encoding": "none",
                        "archive":  "zip",
                        "unpack":   true,
                        "cleanup":  true
                    }
                },
                "service":              "host"
            },

            "meta-compute":  {
                "cmd":      "$execshell $selfpath/$selfexec --pfdcm 10.17.24.163:5015 --PACSservice orthanc --pfurlQuiet --priorHitsTable results.json --indexList 1,2,3  /share/incoming /share/outgoing",
                "auid":     "rudolphpienaar",
                "jid":      "2",
                "threaded": true,
                "container":   {
                        "target": {
                            "image":            "fnndsc/pl-pacsretrieve",
                            "cmdParse":         true
                        },
                        "manager": {
                            "image":            "fnndsc/swarm",
                            "app":              "swarm.py",
                            "env":  {
                                "meta-store":   "key",
                                "serviceType":  "docker",
                                "shareDir":     "%shareDir",
                                "serviceName":  "2"
                            }
                        }
                },
                "service":              "host"
            }
        }
'
```

## Debugging without using containers

Running all the services containerized can result in a lag in debugging, mostly because log files sometimes need to be fully flushed. At times, it is better to run the services non-containerized. 

In such instances, add a breakpoint using ``pudb.set_trace()`` typically in ``charm.py``. Then start the CUBE dev environment in a containerized fashion:

```bash
sudo rm -fr /hostFS/storeBase/* ; sudo rm -fr /usr/users/* ; *make*
```
When execution stops at the breakpoint, kill all the ancillary containers

```bash
dkrm pfcon
dkrm pfioh
dkrm pman
```

where ``dkrm`` is actually a function I have in ``.bashrc``

```bash
dkrm ()
{
    NAME=$1;
    ID=$(dkl | grep $NAME | awk '{print $1}');
    docker stop $ID && docker rm -vf $ID
}
```

and then restart these services directly as per instructions above.



