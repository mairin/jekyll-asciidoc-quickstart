# Debugging CUBE Containerized Services

## Abstract
CUBE is a complex software infrastructure, comprising the main core backend, as well as a constellation of ancillary services. Usually, if things break, the first place to start debugging is to make sure the ancillary services are all OK. This page documents a workflow that can be useful in debugging these CUBE services ``pfcon``, ``pfioh``, and ``pman`` *within* their respective containers.

## Introduction
Due to the distributed and containerized nature of CUBE, debugging the ancillary services ``pfcon``, ``pfioh``, and ``pman`` can be particularly difficult. An initial strategy is to first debug these services outside of ChRIS in a non-containerized setup. If all works "outside" of ChRIS, then a next step is running the services in containers and talking to them from another container, and finally, calling the services via CUBE.

The workflow is to start the whole CUBE system containerized. Then, once all the services are up, to pause and take down each service in turn and restart in an interactive mode. This allows for much more responsive console response, and, if a source file mapping is effected into the container (see below), allows for interactive debugging.

In order to fully debug, the source code of the particular service needs to be volume mapped into its respective container in the correct place. This necessitates small transitory changes to the ``docker-compose.yml``.

## Preconditions

### Map the service source repo tree into the relevant container

Let's assume we want to debug the ``pman`` source code in a running active ``pman`` container. Further, let's assume we have checked out the ``pman`` repository at the same tree level as the ``ChRIS_ultron_backend`` repo. Now, in the ``docker-compose.yml`` add the following line to the ``volumes:`` section of the ``pman_service``:

#### pman
```bash
      - ../pman/pman/pman.py:/usr/local/lib/python3.8/dist-packages/pman/pman.py
```

Repeat for other services if/as necessary

#### pfioh
```bash
    - ../pfioh/pfioh/pfioh.py:/usr/local/lib/python3.8/dist-packages/pfioh/pfioh.py
```

#### pfcon

```bash
    - ../pfcon/pfcon/pfcon.py:/usr/local/lib/python3.8/dist-packages/pfcon/pfcon.py
```

Note that though these service comprise many files (as witnessed in their respective repos), for the overwhelming majority of debug cases you will be interested in only one file per service as shown above.

### When did want to break?

The main thing to resolve first, is *when* in the `CUBE` flow did you want to monitor services? To monitor logs in real time essentially involves stopping and restarting the ancillary services, `pfcon/pman/pfioh`. Should you wish to debug and monitor the integration tests, then `CUBE` should be paused (using a `-p` flag) before the tests and these services restarted before the test execute. However, restarting the services at this point will mean that once the tests are concluded, the final restarting of `CUBE` itself will fail. This failure will occur when the `make.sh` concludes instantiation by restarting `chris:dev` in interactive mode. Such a restart of `chris:dev` will also trigger an additional restart again of the ancillary services. When these services try to come up a second time they will result in address port collisions.

Thus, if you wish to monitor logs in real time of a running `CUBE`, only restart the `pfcon/pman/pfioh` services once `CUBE` has been fully instantiated.

### Add a break to processing flow

It is important to meaningfully pause/stop the flow of setup execution in CUBE. This should happen *after* the initial containerized services have all been launched but *before* real processing has started. 

If you want to monitor logs in the testing phase, start the `make.sh` with a ``-p`` flag, 

```
./make.sh -p
```

and when execution pauses, to remove and restart the relevant services (see below). 

**NOTE THIS WILL MEAN CUBE WILL NOT FULLY INSTANTIATE AFTER ALL THE TESTS ARE DONE**.

If you want to monitor logs in the instantiated phase, only restart the ancillary services once and only once `CUBE` is instantiated.

### Run the dockerized CUBE

From the CUBE repo dir, run (add a ``-d`` for debugging verbosity)

```bash
./make.sh
```

or, if you have *local* docker images:

```bash
./make.sh local
```

## Debug setup

### Create an "environment"

The simplest way to debug is to open four terminals, say in a grid. In each terminal `cd` to the `CUBE` source repo. 

Each terminal will be used to run a specific service. 

```bash
┌────────────────────┐                               ┌─────────────────────┐
│ terminal 1 -- CUBE │                               │ terminal 2 -- pfcon │
├────────────────────┴─────────────────────────────┐ ├─────────────────────┴────────────────────────────┐
│ $>cd $CUBE_source_repo                           │ │ $>cd $CUBE_source_repo                           │
└──────────────────────────────────────────────────┘ └──────────────────────────────────────────────────┘

┌─────────────────────┐                              ┌─────────────────────┐
│ terminal 3 -- pfioh │                              │ terminal 4 -- pman  │
├─────────────────────┴────────────────────────────┐ ├─────────────────────┴────────────────────────────┐
│ $>cd $CUBE_source_repo                           │ │ $>cd $CUBE_source_repo                           │
└──────────────────────────────────────────────────┘ └──────────────────────────────────────────────────┘
```

Obviously don't type literally `$CUBE_source_repo` in the above. Type the path of where your `CUBE` repo lives. Now, first we have to start up `CUBE` and tell it to pause mid-instantiation:

```bash
┌────────────────────┐                                 
│ terminal 1 -- CUBE │                                 
├────────────────────┴─────────────────────────────────────────┐   
│ $>./unmake.sh ; sudo rm -fr FS ; rm -fr FS ; ./make.sh -i -p │   
└──────────────────────────────────────────────────────────────┘   

                                                     ┌─────────────────────┐
                                                     │ terminal 2 -- pfcon │                             
                                                     ├─────────────────────┴────────────────────────────┐
                                                     │ $>                                               │
                                                     └──────────────────────────────────────────────────┘

┌─────────────────────┐                              ┌─────────────────────┐
│ terminal 3 -- pfioh │                              │ terminal 4 -- pman  │
├─────────────────────┴────────────────────────────┐ ├─────────────────────┴────────────────────────────┐
│ $>                                               │ │ $>                                               │
└──────────────────────────────────────────────────┘ └──────────────────────────────────────────────────┘
```

### Stop services and restart in interactive mode

Let `terminal 1` do its thing. Eventually it will PAUSE. 

```bash
┌──────────────────────────────────────┐
│ Thu 17 Sep 15:44:24 EDT 2020 [titan] │
├──────────────────────────────────────┴─────────────────────────────────────────┐
│                   9: Pause for manual restart of services?                     │▒
├────────────────────────────────────────────────────────────────────────────────┤▒
│Pausing... hit *ANY* key to continue                                            │▒
└────────────────────────────────────────────────────────────────────────────────┘▒
 ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒
```

Now we are going to use `docker-compose` to stop and restart each of our ancillary services in their respective terminals in interactive mode. This allows us to have realtime log monitoring. There is a specific order that is required.

First we stop/restart ``pfcon``. Once it is up, we can then stop/restart ``pfioh`` and  then ``pman`` _in that order_!

### Stop/restart `pfcon`

```bash
                                                     ┌─────────────────────┐                            
                                                     │ terminal 2 -- pfcon │                            
                                                     ├─────────────────────┴────────────────────────────┐
                                                     │ $> ./make.sh -s -r pfcon                         │
                                                     └──────────────────────────────────────────────────┘

```
*WAIT* until `pfcon` is back up....

### Now stop/restart `pfioh` and `pman`

```
┌─────────────────────┐                              ┌─────────────────────┐
│ terminal 3 -- pfioh │                              │ terminal 4 -- pman  │
├─────────────────────┴────────────────────────────┐ ├─────────────────────┴────────────────────────────┐
│ $>./make.sh -s -r pfioh                          │ │ $>./make.sh -s -r pman                           │
└──────────────────────────────────────────────────┘ └──────────────────────────────────────────────────┘
```

## Simulate CUBE calls

### Set some convenience environment variables

```bash
export HOST_IP=$(ip route | grep -v docker | awk '{if(NF==11) print $9}' | head -n 1)
export HOST_PORT=8000
```

### ``hello``

Assuming satisfied preconditions, let's say ``hello`` to ``pfcon``. It will in turn ask each of ``pfioh`` and ``pman`` ``hello`` and return the response.

```bash
pfurl --verb POST --raw \
        --http ${HOST_IP}:5005/api/v1/cmd \
        --httpResponseBodyParse \
        --jsonwrapper 'payload' \
        --msg \
'{  "action": "hello",
    "meta": {
                "askAbout":     "sysinfo",
                "echoBack":      "Hi there!",
                "service":       "host"
            }
}' --quiet --jsonpprintindent 4
```

### Simulate an FS plugin

In this call, be sure that the ``HOST_IP`` env variable is set correctly.

```bash
pfurl --verb POST --raw --http ${HOST_IP}:5005/api/v1/cmd \
        --httpResponseBodyParse --jsonwrapper 'payload'     \
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
                        "path":         "/etc"
                },
                "localTarget": {
                        "path":         "/usr/users/test/foo/feed_90/simplefsapp_90/data",
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
                "cmd":      "$execshell $selfpath/$selfexec /share/outgoing --saveinputmeta --saveoutputmeta --dir ./",
                "auid":     "rudolphpienaar",
                "jid":      "89",
                "threaded": true,
                "container":   {
                        "target": {
                            "image":            "fnndsc/pl-simplefsapp",
                            "cmdParse":         true
                        },
                        "manager": {
                            "image":            "fnndsc/swarm",
                            "app":              "swarm.py",
                            "env":  {
                                "meta-store":   "key",
                                "serviceType":  "docker",
                                "shareDir":     "%shareDir",
                                "serviceName":  "89"
                            }
                        }
                },
                "service":              "host"
            }
        }
'
```

### Simulate a DS plugin

In this call, be sure that the ``HOST_IP`` env variable is set correctly.

```bash
pfurl --verb POST --raw --http ${HOST_IP}:5005/api/v1/cmd \
      --httpResponseBodyParse --jsonwrapper 'payload'     \
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
                        "path":         "/neuro/users/rudolphpienaar/Pictures"
                },
                "localTarget": {
                        "path":         "/home/rudolph/tmp/Pictures",
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
                "cmd":      "$execshell $selfpath/$selfexec --prefix test- --sleepLength 0 /share/incoming /share/outgoing",
                "auid":     "rudolphpienaar",
                "jid":      "89",
                "threaded": true,
                "container":   {
                        "target": {
                            "image":            "fnndsc/pl-simpledsapp",
                            "cmdParse":         true
                        },
                        "manager": {
                            "image":            "fnndsc/swarm",
                            "app":              "swarm.py",
                            "env":  {
                                "meta-store":   "key",
                                "serviceType":  "docker",
                                "shareDir":     "%shareDir",
                                "serviceName":  "89"
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


