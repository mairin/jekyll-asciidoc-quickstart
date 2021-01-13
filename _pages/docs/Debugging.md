### 500
#### Description
Error that is not caught on the server

#### Track it down

#### Open Django server console
1- Log into container running Django.
``` bash
docker exec -ti <container id> /bin/bash
```

2- Stop Djando server process
``` bash
ps aux | grep python
kill -9 <pids>
```

3- Restart Django
``` bash
python -m pdb manage.py runserver 0.0.0.0:8000
> add breakpoints or hit "c" to continue
```

Note that there is new way to get an interactive terminal with the latest `docker-make` script:
``` bash
docker-compose run --service-ports chris_dev
```

#### Catch/Raise it
Classic try/catch scheme.

We only Raise errors from the [rest_framework.exceptions](http://www.django-rest-framework.org/api-guide/exceptions/#exceptions).
``` python
  from rest_framework.exceptions import ParseError
  ...
  try:
    for x in stream_data['template']['data']:
      json_data[x['name']] = x['value']
    except KeyError as e:
      detail = "%s field required. " % e 
       detail += template_valid_str  
       raise ParseError(detail=detail)
```