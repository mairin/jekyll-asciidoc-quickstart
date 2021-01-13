```diff
- <Important Note>
- This tutorial is for OSX and might vary on other platforms.
```
----------
# System Overview
![Pacs Pull System Overview](https://docs.google.com/drawings/d/1AMAeeucestZaFkzbUH6sMyU6pYCHNzk3hKG1TGkmxFI/pub?w=960&h=720)

# Setup the DCM SERVER
* Install orthanc: http://www.orthanc-server.com/
* Update configuration: : `~/work/orthancAndPluginsOSX.stable$ vim configOSX.json`
* Config HTTP server:
```
 51   // Enable the HTTP server. If this parameter is set to "false",
 52   // Orthanc acts as a pure DICOM server. The REST API and Orthanc
 53   // Explorer will not be available.
 54   "HttpServerEnabled" : true,
 55 
 56   // HTTP port for the REST services and for the GUI
 57   "HttpPort" : 8042,
...
```
* Config DICOM server:
```
...
 76   // Enable the DICOM server. If this parameter is set to "false",
 77   // Orthanc acts as a pure REST server. It will not be possible to
 78   // receive files or to do query/retrieve through the DICOM protocol.
 79   "DicomServerEnabled" : true,
 80 
 81   // The DICOM Application Entity Title
 82   "DicomAet" : "ORTHANC",
 83 
 84   // Check whether the called AET corresponds during a DICOM request
 85   "DicomCheckCalledAet" : false,
 86 
 87   // The DICOM port
 88   "DicomPort" : 4242,
...
```
* Config network topology:
```
...
140   // The list of the known DICOM modalities
141   "DicomModalities" : {
142     /**
143      * Uncommenting the following line would enable Orthanc to
144      * connect to an instance of the "storescp" open-source DICOM
145      * store (shipped in the DCMTK distribution) started by the
146      * command line "storescp 2000".
147      **/
148      "chrisultron" : [ "CHRIS-ULTRON-AET", "localhost", 2000 ],
149      "chrisultronlistener" : [ "CHRIS-ULTRON-LIS", "localhost", 10401 ]
...
```
## Start Orthanc instance and upload data through HTTP Server
* Start Orthanc:
`~/work/orthancAndPluginsOSX.stable$ ./startOrthanc.command`

Then go to: `http://localhost:8042/app/explorer.html` and add some DICOM data into the Orthanc instance.

* Test with `PyPX`: `px-echo`
``` 
{'data': '', 'command': '/usr/local/bin/echoscu --timeout 5  -aec CHRIS-ULTRON-AEC -aet CHRIS-ULTRON-AET 192.168.1.110 4242', 'status': 'success'}`
```

* Test with DCMTK: `echoscu localhost 4242
```
<empty response>
```

# Setup the STORAGE SERVER
* Install DCMTK: `http://dicom.offis.de/dcmtk.php.en`
* Install PyPx: `https://pypi.python.org/pypi/pypx`
* Add service: `/$ vim /etc/services`
```
13922 chris-ultron    10401/tcp   # chris ultron dicom listener
13923 chris-ultron    10401/udp   # chris ultron dicom listener
13924 #                           Nicolas Rannou <nicolas@eunate.ch>
```
* Configure the DAEMON: (OSX): `/$ vim /Library/LaunchDaemons/org.babymri.chris-ultron.plist`
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>UserName</key>
    <string>nico</string>
    <key>Label</key>
    <string>org.babymri.chris-ultron</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Library/Frameworks/Python.framework/Versions/3.5/bin/python3</string>
        <string>/Library/Frameworks/Python.framework/Versions/3.5/bin/ptk-listen</string>
    	<string>-t</string>
    	<string>/tmp</string>
    	<string>-l</string>
    	<string>/tmp/log</string>
    	<string>-d</string>
    	<string>/tmp/data</string>
    </array>
    <key>inetdCompatibility</key>
    <dict>
        <key>Wait</key>
        <false/>
    </dict>
    <key>Sockets</key>
    <dict>
        <key>Listeners</key>
        <dict>
            <key>SockServiceName</key>
            <string>chris-ultron</string>
            <key>SockType</key>
            <string>stream</string>
        </dict>
    </dict>
</dict>
</plist>
```

* Load Daemon: `sudo launchctl load -w org.babymri.chris-ultron.plist`

(Unload: `sudo launchctl unload org.babymri.chris-ultron.plist`)

* Test Daemon: `nc localhost 10401` (must hang)

# Using the plugins from ULTRON SERVER
```diff
- Make sure the '/tmp/pacsquery' (or equivalent) repo exists.
- Make sure the '/tmp/host/data' (or equivalent) repo exists.
- Make sure the '/tmp/pacsretrieve' (or equivalent) repo exists.
```

### Pacs Query
`
ubuntu@chris-ultron:~/chris-ultron-backend/chris_backend/plugins/services$ python3 ./pacsquery/pacsquery.py "/tmp/pacsquery"`

* Results: `cat /tmp/pacsquery/success.txt`
```
...
List all the data available on the DICOM Server
...
```

### Pacs Retrieve
`
(chris-ultron) ubuntu@chris-ultron:~/chris-ultron-backend/chris_backend/plugins/services$ python3 ./pacsretrieve/pacsretrieve.py --dataLocation "/tmp/host/data" --seriesUIDS "0" --seriesFile "/tmp/pacsquery/success.txt" /tmp/pacsquery/ /tmp/pacsretrieve/
`

When running the command, the PACS Server will print the following message:
```
W0123 10:03:05.789730 OrthancMoveRequestHandler.cpp:175] Move-SCU request received for AET "CHRIS-ULTRON-LIS"
```

The Plugin will return when the required data has be retrieved: `ls /tmp/pacsretrieve/`
```
2175-Anonimo/<STUDIES>/<SERIES>
```

```diff
- /tmp/pacsretrieve/ might be empty if timer is to short: increase it by increasing the wait time at: https://github.com/FNNDSC/ChRIS_ultron_backEnd/blob/master/chris_backend/plugins/services/pacsretrieve/pacsretrieve.py#L117
```