# Create a new ChRIS user account

### Just make an unauthenticated POST request to the `/api/v1/users/` ChRIS API end-point with the required `username`, `password` and `email` descriptors
For instance, using the [httpie client](https://github.com/jakubroztocil/httpie) and assuming that the ChRIS service is running at `localhost:8000`:

```bash
http POST http://localhost:8000/api/v1/users/ Content-Type:application/vnd.collection+json Accept:application/vnd.collection+json template:='{"data":[{"name":"email","value":"newuser@babymri.org"}, {"name":"password","value":"newuser1234"}, {"name":"username","value":"newuser"}]}'
```