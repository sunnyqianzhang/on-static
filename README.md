# on-static

__`on-static` is a tool that help [RackHD](https://github.com/rackhd) users to
manage static file resources like os images.__

RackHD has an built-in static file server that is used by OS install workflow. An small [setup script](https://github.com/RackHD/on-tools/blob/master/scripts/setup_iso.py) will help users to mount the os images. It's running great as long as the amount of managed nodes is not too many so that the static file server will not consume to much hardware resouces (CPU, memory, etc). 

As long as users trying to scale up the managed nodes, they would think to move the static server out to a seperate hardware. RackHD allows user to use a standalone file server and on-static is meant to simplify the step to setup a standalone static file server, and provide a very easy way of managing os images and iso files. 

Furture support may include management of other static resources
like skupack, microkernel or overlayfs. __This is not implemented so far.__

_Copyright 2015-2016, EMC, Inc._

## Introduction

With `on-static`, users can

* **Create http server** that servers operation system images used for [RackHD](https://github.com/rackhd) OS installation. OS images are mounted from iso files that can be loaded from different sources.
* **Mount uploaded iso file** and expose it from http server.
* **Manage iso** files. Users can list in-store iso files, upload a new iso file, delete a in-store iso file.
* **Restore** settings after service restart. On-static service store user image settings on file-based persistant storage. Evertime on-static service get started, it will load the user image settings from the storage and get everything setup. 

Two http servers will be created by default:

* The northbound: Handles user requests like managing(list/create/delete) an OS image, or managing(list/create/delete) an iso file.
* The southbound: The http file server.

## Installation

Install on-static is quite straight forward.

    git clone https://github.com/cgx027/on-static.git
    cd on-static
    npm install

## Running

    sudo node index.js

The northbound API will by default listen at 0.0.0.0:7070, and the southbound will by default listen at 0.0.0.0:9090. Those IP addresses and ports are user configurable.

## Use it with RackHD

Assuming user already have a RackHD instance running and he/she wantted to install ubuntu to one of his/her nodes. 

First, it will has to setup a unbutu image server. The setups are:

1. Locate the unbutu iso file from somewhere. User can choose to download it by themselves or just ask on-static to download it instead. 
2. Setup the ubuntu image server by send a PUT request to on-static northbound API, something look like:

        curl -X PUT "http://10.62.59.150:7070/images?name=ubuntu&version=14.04&isoweb=http://10.62.59.150:9090/iso/photon-1.0.iso"

    where on-static will help download the iso from the link specified. If user had download the iso by himself/herself, he/she can use following API to upload the image to on-static:
        
        curl -X PUT "http://10.62.59.150:7070/images?name=ubuntu&version=14.04&isoclient=client.iso" --upload-file path-to-file/test.iso

3. Specify repo: "http://on-static-ip-addr:port/ubuntu/14.04" in the payload used in os install workflow. 

    The API look like:

        http://{{host}}/api/2.0/nodes/:identifier/workflows?name=Graph.InstallUbuntu

    And the pay load look like:

    ```
    {
        "options": {
            "defaults": {
                "version": "trusty",
                "baseUrl": "install/netboot/ubuntu-installer/amd64",
                "kargs": {
                    "live-installer/net-image": "http://on-static-ip-addr:port/ubuntu/14.04/ubuntu/install/filesystem.squashfs"
                },
                "repo": "http://on-static-ip-addr:port/ubuntu/14.04"
            }
        }
    }
    ```

## API

### Northbound

* Image management

    1. GET http://0.0.0.0:7070/images: Get/list all OS images. No parameter is needed.

        ```
        curl http://0.0.0.0:7070/images | python -m json.tool
        {
            "act": "Listing all images",
            "images": [
                {
                    "id": "6807e9b2-a763-4b74-bfe9-4b20fe964400",
                    "iso": "client.iso",
                    "name": "photon",
                    "status": "OK",
                    "version": "6.0"
                }
            ],
            "message": "Received get request for listing images",
            "query": {}
        }
        ```

    2. PUT http://0.0.0.0:7070/images: Add OS images. Three parameters are needed. 
        * name: in query or body, the OS name. Should be one of [ubuntu, rhel, photon, centos]. The list will expand as we move on. 
        * version: in query of body, the OS version. User can use any string that they like. Examples will include, 14.04, 14.04-x64, 14.04-EMC-Ted-test.
        * isoweb, isostore, isolocal or isoclient: the source of the iso file used to build the OS image. At least one of those four should be specified. If more then one is specified, on-static will use isostore over isolocal over isoweb over isoclient. 
            * isoweb: use iso file from web, can be http or ftp, like _http://example.com/example.iso_. **Https** is not tested yet.
            * isolocal: use iso file from the server which on-static in running.
            * isoclient: use iso file uploaded from user client where the APIs are called. Iso files are uploaded using HTTP PUT method.
            * isostore: use in-store iso file that had been uploaded before from above three sources. This is useful when you are adding a OS image that had been removed earlier.
        
            Using **isoclient**

            ```
            curl -X PUT "http://10.62.59.150:7070/images?name=photon&version=1.0&isoclient=client.iso" --upload-file path-to-file/test.iso
            Uploaded 100 %
            Upload finished!
            {
                "message": "Received put request for create images",
                "query": {
                    "name": "photon",
                    "version": "1.0",
                    "isoclient": "client.iso"
                },
                "body": { },
                "act": "Adding images for os named photon version 1.0",
                "images": [
                    {
                        "id": "3065063b-d993-471b-8573-e8afcfb713fa",
                        "iso": "client.iso",
                        "name": "photon",
                        "version": "1.0",
                        "status": "preparing"
                    }
                ]
            }
            ```

            Using **isoweb**. image status will set as 'downloading iso' and iso download will be carried out at the background. You can check the status afterwards using get/list image API. 

            ```
            curl -X PUT "http://10.62.59.150:7070/images?name=photon&version=1.0&isoweb=http://10.62.59.150:9090/iso/photon-1.0.iso"
            {
                "message": "Received put request for create images",
                "query": {
                    "name": "photon",
                    "version": "1.0",
                    "isoweb": "http://10.62.59.150:9090/iso/photon-1.0"
                },
                "body": { },
                "act": "Adding images for os named photon version 1.0",
                "images": [
                    {
                        "id": "39647624-e640-41d0-901b-afc58af98725",
                        "iso": "photon-1.0.iso",
                        "name": "photon",
                        "version": "1.0",
                        "status": "downloading iso"
                    }
                ]
            }
            ```

            Using **isolocal**. 

            ```
            curl -X PUT "http://10.62.59.150:7070/images?name=centos&version=7.0&isolocal=/home/onrack/github/on-static/static/files/iso/centos-7.0.iso"
            {
                "message": "Received put request for create images",
                "query": {
                    "name": "centos",
                    "version": "7.0",
                    "isolocal": "/home/onrack/github/on-static/static/files/iso/centos-7.0.iso"
                },
                "body": { },
                "act": "Adding images for os named centos version 7.0",
                "images": [
                    {
                        "id": "9fce7e8f-c7ef-49db-a47f-1924675d5e29",
                        "iso": "/home/onrack/github/on-static/static/files/iso/centos-7.0.iso",
                        "name": "centos",
                        "version": "7.0",
                        "status": "preparing"
                    }
                ]
            }
            ```

            Using **isostore**.

            ```
            curl -X PUT "http://10.62.59.150:7070/images?name=centos&version=7.0&isostore=centos-7.0.iso"
            {
                "message": "Received put request for create images",
                "query": {
                    "name": "centos",
                    "version": "7.0",
                    "isostore": "centos-7.0.iso"
                },
                "body": { },
                "act": "Adding images for os named centos version 7.0",
                "images": [
                    {
                        "id": "b6b3e3be-c799-4af4-86c8-09a99d3aa7c7",
                        "iso": "centos-7.0.iso",
                        "name": "centos",
                        "version": "7.0",
                        "status": "preparing"
                    }
                ]
            }
            ```

    3. DELETE http://0.0.0.0:7070/images: delete an OS images. two parameters are needed. 
        * name: in query or body, the OS name. 
        * version: in query of body, the OS version.

        ```
        curl -X DELETE -H "Content-Type: application/json" -d '' "http://10.62.59.150:7070/iso?name=client.iso"
        {
            "message": "Received request for deleting iso files",
            "query": {
                "name": "client.iso"
            },
            "iso": [
                {
                    "name": "centos-7.0.iso",
                    "size": "4.15 GB",
                    "upload": "2016-10-18T18:02:50.769Z"
                },
                {
                    "name": "test.iso",
                    "size": "1.00 KB",
                    "upload": "2016-10-21T10:02:01.204Z"
                }
            ]
        }
        ```

* Ios file management

    1. Get/list install iso files.

        ```
        curl -X GET "http://10.62.59.150:7070/iso" 
        {
            "iso": [
                {
                    "name": "centos-7.0.iso",
                    "size": "4.15 GB",
                    "upload": "2016-10-18T18:02:50.769Z"
                },
                {
                    "name": "test.iso",
                    "size": "1.00 KB",
                    "upload": "2016-10-21T10:02:01.204Z"
                }
            ],
            "message": "Received get request for listing iso files",
            "query": {}
        }
        ```

    2. Upload an iso file. One parameter is needed.
        * name: in query or in body, the name of the iso that will be shown in the store. 

        ```
        curl -X PUT "http://10.62.59.150:7070/iso?name=test.iso" --upload-file static/files/iso/centos-7.0.iso
        Uploaded 10 %
        Uploaded 20 %
        Uploaded 30 %
        Uploaded 40 %
        Uploaded 50 %
        Uploaded 60 %
        Uploaded 70 %
        Uploaded 80 %
        Uploaded 90 %
        Uploaded 100 %
        Upload finished!
        ```

    3. Delete a iso file that is in the store. One parameter is needed.
        * name: in query or in body, the name of the iso will be deleted. 

        ```
        curl -X DELETE "http://10.62.59.150:7070/iso?name=test.iso"
        {
            "message": "Received request for deleting iso files",
            "query": {
                "name": "test.iso"
            },
            "iso": [
                {
                    "name": "centos-7.0.iso",
                    "size": "4.15 GB",
                    "upload": "2016-10-18T18:02:50.769Z"
                }
            ]
        }
        ```

### Southbound

The southbound is all about static file server. It's by default listen at 0.0.0.0:9090. It also expose a GUI so that if you navigate to http://0.0.0.0:9090/ using your favorate web browser, you will get files and directories listed. 

## Configure

There are not much to be configured for on-static. The Configuration is set on on-static/config.json file. Following is an example:

```
{
  "httpEndpoints": [
    {
      "address": "0.0.0.0",
      "port": 7070,
      "routers": "northbound"
    },
    {
      "address": "0.0.0.0",
      "port": 9090,
      "routers": "southbound"
    }
  ],
  "httpFileServiceRootDir": "./static/files",
  "httpFileServiceApiRoot": "/",
  "isoDir": "./static/files/iso",
  "inventoryFile": "./config.json",
  "images": []
}
```

The Configurations explained as below:

* httpEndpoints: the http endpoint settings. Each endpoint represent a http service, eight northbound or southbound. At lease one endpoint for northbound service and one endpoint for southbound service is a must have. More endpoints are also supported as per user configuration needs. Each endpoint has three parameters:
    * address: the IP address that the service is listen on. Specially, 0.0.0.0 means by listen on all network interfaceses, 127.0.0.1 means only listen to local loop interface. 
    * port: the IP address that the service is listen on.
    * routers: should be one of northbound and southbound. 

    Care should be taken when configuring the endpoints to makesure the IP address and port is not conflicting with other web services on the same server. 

* httpFileServiceRootDir: the root dir that the sourcebound service will serve. It should be a relative path to the on-static root directory. Furture work can be added to support absolute path. 
* httpFileServiceApiRoot: the API root for southbound service. 
* isoDir: the dir where user uploaded iso files will be stored. Also a relative path.
* inventoryFile: the file where user image settings are stored. This should ONLY be set to ./config.json' by now but can be refactored to be other files. 
* images: the user image settings. Updated as per user calls southbound APIs. 


## Contributions are welcome