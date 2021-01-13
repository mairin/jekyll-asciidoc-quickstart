# Workflow notes

## Abstract

This page describes a simple workflow in the ChRIS Ultron backend system. The workflow consists of getting a list of plugins, calling a 'simplefs' plugin, checking its status, calling a 'simpleds' plugin on this 'simplefs' output, and checking status.

## Preconditions

### service assumptions

We assume that the django and pman services are running on localhost (or 10.17.24.163) and listening on ports 8000 and 5010 respectively.

### source the python virtual environment

_Note: your mileage may vary! Adjust the following where/as necessary for your local setup. These notes are accurate for Rudolph's setup!_

First, source the python virtual environment

```bash
export WORKON_HOME=~/src/python-env
source /usr/local/bin/virtualenvwrapper.sh
```

and `workon` the relevant project:

```bash
workon chris_env
cd ~/src/ChRIS_ultron_backend/chris_backend
```

### startup the django server

```bash
manage.py runserver localhost:8000
```

### startup the `pman` server

```bash
pman --rawmode 1 --http --port 5010 --listeners 12
```

## Preliminar interaction

### GET the homepage

```bash
purl  --auth chris:chris1234                                    \
        --content-type application/vnd.collection+json          \
        --verb GET                                              \
        --raw --jsonwrapper template                            \
        --http localhost:8000/api/v1/                           \
        --quiet | underscore print --color
```

### GET an arbitrary plugin instance

```bash
purl  --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb GET                                              \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/instances/90/   \
        --quiet | underscore print --color
```

## Workflow 

### GET plugin list

```bash
#0. GET plugin list
purl    --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb GET                                              \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/                \
        --quiet | underscore print --color
```

### Call a `simplefs` plugin

```bash
# 1. Call a 'simplefs' plugin
purl    --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb POST                                             \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/1/instances/    \
        --msg \
        '{ 
            "data": [{  "name":     "dir",                
                        "value":    "/home"}] 
         }' --quiet  | underscore print --color
```

### Check the status of the `simplefs` call

```bash
# 2. Check status
purl    --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb GET                                              \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/instances/399/  \
        --quiet | underscore print --color
```

### Call a `simplefs` plugin on the `simplefs` output

```bash
# 3. Call a 'simpleds' plugin on 'simplefs' output
purl    --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb POST                                             \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/5/instances/    \
        --msg \
        '{ 
            "data": [{  "name": "prefix",           "value": "test-"}, 
                     {  "name": "sleepLength",      "value": "30"},
                     {  "name": "previous_id",      "value": "401"}] 
         }' --quiet  | underscore print --color
```

### Check the final status

```bash
# 4. Check status
purl    --auth chris:chris1234                                  \
        --content-type application/vnd.collection+json          \
        --verb GET                                              \
        --raw --jsonwrapper template                            \
        --http 10.17.24.163:8000/api/v1/plugins/instances/404/  \
        --quiet | underscore print --color
```


