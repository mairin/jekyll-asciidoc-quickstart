### Query
Command to run a PACS_Query:

``` bash

python3 pacsquery.py \n
  --aec ORTHANC --aet CHIPS \n
  --serverIP 192.168.1.10 --serverPort 4242 \n
  --patientID "2491"  --patientName "" --patientSex "" \n
  --studyDescription "" --studyDate "" --modalitiesInStudy "" \n
  --seriesDescription "" \n
   --performedStationAETitle ""  \n
  /tmp/query/ #output directory

```

That will create a new file “success.txt” in the output directory (`/tmp/query`).


* **Caveats #1**: All parameters are not connected for search yet (i.e. `studyDate` has no effect) (edited)
* **Caveat #2**: If we provide multiple patient IDs, it only uses the first one. (edited)
(if patientID = “”, it will search against all Patients)

### Retrieve
Command to run a PACS_Retrieve:

 ```bash

python3 pacsretrieve.py \n
  --aec ORTHANC --aet CHIPS --aetListener CHIPS \n
  --serverIP 192.168.1.110 --serverPort 4242
  --dataLocation /incoming/data/ \n
  --seriesUIDS "0,1,2,3,4,5,6" \n
  /tmp/query/ /tmp/retrieve/ # input and ouput directories
```

1. Fetch `success.txt` from `/tmp/query`
2. Retrieve selected `seriesUIDS` from this file.
3. Returns once the data has been received in `dataLocation`.

The PacsServer (ORTHANC) must push the data to a `dicom_listener` that will pack the data into `/incoming/data`.
`aetListener` is used to let ORTHANC know where to push data to.

Useful resources:

* https://github.com/FNNDSC/pypx/wiki/dicom_listener
* https://github.com/FNNDSC/ChRIS_ultron_backEnd/wiki/PACS-Pull-wokflow