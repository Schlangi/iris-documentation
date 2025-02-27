

# How to connect your App to IRIS


In this document we describe how apps can be connected to the IRIS system.  


## Overview

The IRIS System consists of different actors that form trust relationships with each other based on certificates. These actors are 1) IRIS Central services 2) Contact Tracing Applications (you) and 3) Health Departments. Actors use the [IRIS endpointserver](https://github.com/iris-gateway/eps) in order to exchange messages with other actors. Each actor is registered in the public IRIS service directory alongside with its corresponding role on the IRIS system.

Please find below and overview of the IRIS system and its actors as well as the flows that are relevant for your application. 

![Overview IRIS System](iris-system.jpg)

|Step|Description|
|-|-|
|1|You need to send a certificate signing request (csr) to the [IRIS rollout team](mailto:rollout@iris-gateway.de). The rollout team will provide you with a certificate in return. You need to configure your EPS to use that certificate.|
|2|The IRIS rollout team registers your app and the endpoint domain in the service direcory.|
|3|Your app pushes the locations for those it is responsible for into the central locations service. This is a frequent process and depends on the lifecycle of the locations that are managed within by app|
|4|The health authority employees can use the IRIS client (having ints own EPS) in order to search for locations.|
|5|The IRIS client connects to your app through the EPS layer and requests guest lists for a specific date and time. In order to know the endpoint IRIS internally keeps a record of which location is owned by which app. Your app returns the contact information using EPS.|

---

### Prerequisites

This guide will show step by step technically integrating an app to IRIS connect.
- [Certificate generation and enrollment](#1-certificate-generation-and-enrollment)
- [Local deployment of the EPS](#2-install-and-configure-eps)
- [General usage of the EPS](#3-interacting-with-eps)
- [Services and Methods of the EPS](#32-destinations-and-methods)
- [Use EPS to post locations to the IRIS connect search index](#41-location-search-index)
- [Listen to data requests of the health department submitted to the EPS](#42-process-iris-client-data-requests)
- [Transmit requested data to the health department using EPS](#43-send-data-submissions)

You need to open port 4444, 4443 or 443 for incoming connections for test environment. 

The EPS installed on your server needs to be reached on port 4444, 4443 or 443 for test environment.

You will need openssl to create the certificate and preferably docker to run the EPS.

---

## 1 Certificate Generation and Enrollment

A unique certificate for your app is needed to achieve trust for communication reaching out from your local EPS to remote EPS of IRIS central services or IRIS clients.

The certificate is signed by IRIS connect with a root certificate. The root certificate itself delivers trust for the local EPS for incoming communication.

### 1.1 Generate Certificate Signing Request

Generate your certificate signing request (csr) using openssl for the IRIS Connect test server.

Please use your app name as CN (for example CN=smartmeeting). *Don't use spaces and use only lower-case*.

    O="COSYNUS GmbH"
    ST="Europaplatz 5"
    L="64293 Darmstadt"
    C="DE"
    OU="IT"
    CN=[yourappname]
    # using less than 1024 here will result in a TLS handshake failure in Go
    # using less than 2048 will cause e.g. 'curl' to complain that the ciper is too weak
    LEN="2048"


    openssl genrsa -out "[yourappname].key" 4096;
  	openssl rsa -in "[yourappname].key" -pubout -out "[yourappname].pub";
  	openssl req -new -sha256 -key "[yourappname].key" -subj "/C=${C}/ST=${ST}/L=${L}/O=${O}/OU=${OU}/CN=${CN}" -addext "subjectAltName = DNS:[yourappname],DNS:*.[yourappname].local" -out "[yourappname].csr";

### Production Certificate Signing Request

The creation of the CSR for production is the same as the one for test. Please don't use the exact same CSR. You need to generate another private key.

### 1.2 Request the Certificate

Send the .csr and your domain to [IRIS rollout team](mailto:rollout@iris-gateway.de) and get your .crt file back from us.

Please make sure to tell us, which environment (test or prod) you are requesting.

Example
    
    Hi IRIS Team,

    I provide the csr for api.test.perkiot.com:4444 (test).

    BR
    Mic
    
### 1.3 Legal requirements for production

Each app provider must go through a process to be connected to the production system.

In addition to the legal requirements, for which we update the details promptly, it is important to be able to demonstrate a fully functional connection to the test system.

## 2 Install and configure EPS

The EPS (IRIS EndPointService) does all the magic in a black box:
- encryption of the data channel to the IRIS clients requesting the contact lists
- ensure trust for IRIS and your app
- dealing with p2p networking to eliminate bottlenecks

To be able to act, the EPS needs to be configured and trusted, which means it needs the trusted and unique certificate signed from the [csr](#11-generate-certificate-signing-request) you have send to IRIS team, and it must be made known to IRIS, that is the step IRIS team does with the provided URI before issuing the certificate.

### 2.1 Artifacts needed to start

You need the following:
- your [certificate](#11-generate-certificate-signing-request)
- docker image inoeg/app-eps

### 2.2 Local Settings

The EPS config is mainly done by ENVIRONMENT variables. Please create a directory that holds your configuration. For example `/data/eps/`. Please copy the `.env.sample` file as `.env` to this local path.

The certificate issued by IRIS based on your signing request and the corresponding key must be stored in the certs folder. In our example the certificate and key would be stored in `/data/eps/certs`.

You will then need to adapt the `.env` file to your environment.

`APP_CN` must be set to the common name you used for your certificate. Also the certificate .key and .crt files need to be named accordingly. 

`APP_BACKEND_ENDPOINT` specifies the endpoint of your local JSON-RPC server.

`EPS_SETTINGS` is set to `settings/roles/test/app` for test environment or `settings/roles/prod/app` for production. Caution: Do not replace `app` in this string. It's meant to be `app`.
        
### 2.3 Start your EPS

If you are on Linux command line and defined it above, you could simply use the statement as is.

You can start a local eps with

    docker run --name iris-eps -p 5556:5556 -p 4444:4444 -v "/data/eps/certs:/app/settings/certs" :ro --env-file .env inoeg/app-eps:latest

Port 4444, 4443 or 443 are valid for test environment. 

Ports 4444 and 443 can be used for the production environment.

You can change port 5556 to your needs. Requests of your app, that shall reach out to IRIS Connect, will then be sent to `POST https://localhost:5556/jsonrpc` (or your local URL within your own network).

To change the ports, simply replace the port before the colon.

## 3 Interacting with EPS

The EPS is now locally reachable for your app, and it is publically reachable for IRIS Connect.
Let's test, if we are able to reach IRIS Connect from EPS.

### 3.1 Sending messages

Sending messages to IRIS is done with JSON-RPC. 

    {
        "method": "[destination-identifier].[methodname]", 
        "id": "1", 
        "params": {
            "param1": "test"
        }, 
        "jsonrpc": "2.0"
    }
    
You can find the specification of the respective methods and destinations [here](#destinations-and-methods).

To test, we will simply send a test location to IRIS test:

    curl --header "Content-Type: application/json" --request POST --insecure --data '
    {
        "method":"ls-1.postLocationsToSearchIndex",
        "id": "1",
        "params": {
            "locations": [
                {
                    "id": "1abc",
                    "name": "Visits Test"
                }
            ]
        },
        "jsonrpc": "2.0"
    }
    ' https://localhost:5556/jsonrpc

We should get 

    {
        "jsonrpc" : "2.0",
        "result" : "OK",
        "id" : "1"
    }

It is important to know, what is going wrong if you won't get this response. If you post your request to a wrong endpoint, you would just get a "message":"not found". If the port 5556 is not bound to the eps container, you will get a "connection refused".

| cURL result | Possible cause | Ckecks to perform |
|-|-| - |
| Client sent an HTTP request to an HTTPS server. | You called the endpoint at http:// instead of https:// | Ckeck the address in cURL command and options you used. |
| Connection refused | The EPS is not listening on the URI or port of the request. | Ckeck the address in cURL command. Are you running the command from localhost - the docker host, or do you need to address an IP address in your network? Is there a firewall blocking the port? Is the EPS running - check it with docker ps, and check the logs. |
| "message":"not found" | You addressed wrong endpoint | If you address https://localhost:5556/ (without endpoint address jsonrpc) you will get this result. |
| {"jsonrpc":"2.0","error":{"code":-32603,"message":"internal error"},"id":"1"} | There is a bug in EPS or your configuration is incorrect. | Check the settings yml, try another EPS version. |
| {"jsonrpc":"2.0,","error":{"code":-32700,"message":"JSON required"},"id":null} | Your JSON content is malformed. | Check your JSON string as it is passed to the http request's body, maybe try exactly this string with cURL to debug your content. |
| {"jsonrpc":"2.0,","error":{"code":-32600,"message":"invalid request","data":{"message":"invalid input data: params(not a map)","code":"FORM-ERROR","data":{"params":"not a map"},"traceback":[]}},"id":null} | The JSON content is not complete or contains mismatched properties. | In the given example, the property params was spelled wrong (uppercase Params), so check exact matches of the request properties. |

This list is propably not complete. Especially internal errors will have many possible causes. In case you get some internal errors, please check twice if your settings are correct, the EPS is running in level trace and have a look on log messages.

We will update the list of possible errors as they come around.

### 3.2 Destinations and methods

|Destination|Methodname|Annotations|
|-|-|-|
| ls-1 | [postLocationsToSearchIndex](#411-postlocationstosearchindex) | Publish managed locations to IRIS connect, for which health departments may request contact data.
| ls-1 | [getProviderLocations](#412-getproviderlocations) | Verify which locations managed by your app are published to IRIS connect.
| ls-1 | [deleteLocationFromSearchIndex](#413-deletelocationfromsearchindex) | Remove a managed location if the location canceled using your app etc.
| - | [createDataRequest](#42-process-iris-client-data-requests) | This method has to be an endpoint of your app to receive the details for an data request.
| hd-xyz | [submitGuestList](#431-send-data-from-app-backend) | After getting a data request, the data is send to the hd-endpoint given in _client.name in the request.


## 4 Integration of EPS Methods in your App

As described above, you have to 
- publish locations managed by your app
- receive data requests
- submit data

### 4.1 Location search index

Location search index can be reached at `ls-1`.

### 4.1.1 postLocationsToSearchIndex

##### Request

Example parameters:

    "params": {
            "locations": [
                {
                    "id": "5eddd61036d39a0ff8b11fdb",
                    "name": "Restaurant Alberts",
                    "contact": {
                        "officialName": "Meine RestoGruppe GmbH",
                        "representative": "Max Mustermann",
                        "address": {
                            "street": "Türkenstr. 7",
                            "city": "München",
                            "zip": "80333"
                        },
                        "email": "covid@restaurant.de",
                        "phone": "0151 47110815"
                    }                
                }
            ]
        }
        
| Parameter | Description | Annotations |
| --- | --- | --- |    
| `locations` | array of locations | 

Location parameters:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `id` | your unique location id | Identifies the location in your system. Will be included in incoming requests 
| `name` | location friendly name | You could also split your clients sites here. Example: client has site "Hauptstrasse" and "Kaiserstrasse" could be split in two locations with their own location id.
| `contact` | contact information | see below

Contact parameters:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `officialName` | locations official name |  
| `representative` | locations representative | 
| `email` | email | 
| `phone` | phone number

##### Response 

    "result": {
            "_": "OK"
        },

`OK` if successful. Errormessages to be done.

#### 4.1.2 getProviderLocations

Example parameters:
    
    "params": {                  
    }

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `providerId` | your unique ProviderId | Caution: this will change and will not be required in the future.

##### Response 

    "result": {
            "_": [
                {
                    "id": "5eddd61036d39a0ff8b11fdb",
                    "name": "Restaurant Alberts"
                }
            ]
    }

Array of `id` and `name` pairs, where id is your unique identfier and `name` the locations friendly name. Errormessages to be done. 

#### 4.1.3 deleteLocationFromSearchIndex

Example parameters:

    "params": {
        "locationId":"5eddd61036d39a0ff8b11fdb"
    }

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `locationId` | your internal location identifier | | 

##### Response 

    "result": {
            "_": "OK"
        },

`OK` if successful. `NOT FOUND` if location did not exist. Errormessages to be done.

## 4.2 Process IRIS Client data requests

You need to provide a json-rpc endpoint on your server. The exact endpoint can be configured in your EPS configuration file. 
    
    - name: main JSON-RPC client # creates outgoing JSONRPC connections to deliver and receive messages
        type: jsonrpc_client
        settings:
          endpoint: http://[your-host]:[your-port]/[your-endpoint]
          
To receive incoming DataRequests, you need to implement a method `createDataRequest` on your Endpoint. The following request is an example the request generated on https://client.test.iris-gateway.de - the parameter _client.entry is left out here and contains certificates and fingerprints as well as provided services at the endpoint.

```
{
  "jsonrpc": "2.0",
  "method": "createDataRequest",
  "params": {
    "_client": {
      "entry": {[...]},
      "name": "hd-1"
    },
    "dataRequest": {
      "dataAuthorizationToken": "8f4b3c3a-61e1-4e87-954e-5c6ba4f0e051",
      "end": "2021-06-09T16:00:00Z",
      "locationId": "d1abc",
      "proxyEndpoint": "d9e2f1c1-ee8d-48e2-be03-e36fb647222d.proxy.test-gesundheitsamt.de",
      "requestDetails": "",
      "start": "2021-06-09T09:00:00Z"
    }
  },
  "id": "visits.createDataRequest(44693335)"
}
```

The method has to accept the following paramters:

createDataRequest parameters:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `_client` | Calling EPS details |  
| `dataRequest` | DataRequest details |

_client object:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `name` | Calling eps | You will use this name to reach the endpoint for data submissions |
| `entry`| Signatures | Certificates and fingerprints to ensure authenication of the request, will be handled by your EPS and you may ignore this. | 

dataRequest object:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `start` | Start of requested time period | 
| `end` | End of requested time period |  
| `requestDetails` | Additional message (optional) | May contain additional information about the request   
| `dataAuthorizationToken` | Identifies the request to the health department | Used in submission to ensure that the data has been requested by the HD.   
| `proxyEndpoint` | Identifies the connection to public proxy | Neccessary to be able to send submissions from the browser.
| `locationID` | Your location identifier | The identifier you used when creating the location.

Response:

| Parameter | Value | 
| --- | --- |  
| `_` | ``OK`` | 

The request can be saved and edited locally. The functions for transmitting the data to the health authorities are the next step, but asynchronously. This means, there is no need to collect data and post the response within a short timeframe.

It seems to be a good idea to store all requests created an if they are answered already, also to have a possibility for security audits. If you already store actions of exporting CSV file data for health authorities in your app, you might just add some fields to that table.

## 4.3 Send data submissions

There are two ways to submit data to IRIS. The two ways result from the two different types of data management.

Apps that can provide data unencrypted in the backend can send data directly to IRIS via EPS from the backend. Deliver your data to your local EPS after collection.

If the user data must first be decrypted by the operator or another authority in the browser, it can then be transmitted directly from the browser via end-to-end encryption. In this case the data submission has to be posted to the URL provided in the createDataRequest parameter `proxyEndpoint`. For this scenario, you add a "_client":{"name":"$APP_NAME"} to the params, where $APP_NAME is the name for your app as used in the "CN" field of your certificate from the [csr](#11-generate-certificate-signing-request).

__Please note, that the timeframe in the request (startDate to endDate) has to be evaluated as: attendFrom is earlier than endDate AND attendTo is later than startDate. This way all visits that overlap start or end are included.__ In full pseudocode, the boolean evaluation should be like `Visit.Entry <= Request.End && Visit.Exit >= Request.Start`.

### 4.3.1 Parameters for data submissions


    "params":{
        "_client": {
            "name": "mydemoapp"
        },
    	"dataAuthorizationToken": "2edd34d6-bc7b-11eb-8529-0242ac130003",
    	"guestList": {
    		"dataProvider": {
    			"name": "SmartMeeting",
    			"address": {
    				"street": "Europaplatz",
    				"houseNumber": "5",
    				"zipCode": "64293",
    				"city": "Darmstadt"
    			}
    		},
            "startDate": "2021-05-18T10:00:00.000Z",
            "endDate": "2021-05-19T10:00:00.000Z",
            "additionalInformation": "",
    		"guests": [
    			{
    				"firstName": "Hans",
    				"lastName": "Müller",
    				"sex": "UNKNOWN",
    				"email": "p5o50dtktj@temporary-mail.net",
    				"phone": "0151 47110815",
    				"mobilePhone": "0151 47110815",
    				"address": {
    					"street": "Lietzensee-Ufer",
    					"houseNumber": "75",
    					"zipCode": "01657",
    					"city": "Meißen"
    				},
    				"attendanceInformation": {
    					"attendFrom": "2021-03-28T19:21:28.071Z",
    					"attendTo": "2021-03-28T19:21:28.071Z",
    					"additionalInformation": "Tisch 4"
    				}
    			}
    		]
    	}
    }
      
Parameters:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `_client` | Identifies the sending client | This is optional for sending from backend, but needed when sending from browser client. |
| `dataAuthorizationToken` | Authorizes data submission | 
| `guestList` | Guest list information |

_client object:

| Parameter | Description | Annotations |
| --- | --- | --- |    
| `name` | CN from cert | Your app is identified by this name, taken from the certificate's CN |

GuestList object:

| Parameter | Description | Required | Annotations |
| --- | --- | --- | --- |   
| `startDate` | GuestList starts | true | As given in the createDataRequest in time format "yyyy-MM-ddTHH:mm:ssZ" |  
| `endDate` | GuestList ends | true | As given in the createDataRequest in time format "yyyy-MM-ddTHH:mm:ssZ" | 
| `additionalInformation` | Additional information | true | Can be left empty for now - is not displayed |
| `dataProvider` | DataProvider object | true | This is identifying you as the app provider! |
| `guests` | List of guest objects | true | Could of cause be empty, but must be provided |

DataProvider object:

| Parameter | Description | Required | Annotations |
| --- | --- | --- | --- |   
| `name` | Your name | true |   
| `address` | Address object | true |

Guest object:

| Parameter | Description | Required | Annotations |
| --- | --- | --- | --- |   
| `firstName` | First name | true |   
| `lastName` | Last list | true |
| `sex` | Sex | false | `[ MALE, FEMALE, OTHER, UNKNOWN ]` 
| `email` | E-Mail | false |
| `phone` | Phone number | false |
| `mobilePhone` | Mobile number | false |
| `address` | Address object | false |
| `attendanceInformation` | Attendance object | true |

Address object:

| Parameter | Description | Required | Annotations |
| --- | --- | --- | --- |       
| `street` | Street | false |   
| `houseNumber` | House number | false |
| `zipCode` | Zip code | false | 
| `city` | City | false |

AttendanceInformation object:

| Parameter | Description | Required | Annotations |
| --- | --- | --- | --- |       
| `attendFrom` | Attend from | true | Time format "yyyy-MM-ddTHH:mm:ssZ"  
| `attendTo` | Attend to | true | Time format "yyyy-MM-ddTHH:mm:ssZ"
| `additionalInformation` | Additional attendance information | false | For example table or area 

### 4.3.2 Send data from app backend

The data is sent to the custom EPS via JSON-RPC. The method name used is `[hdEndpoint].submitGuestList`. [hdEnpoint] corresponds to _client.name from the received DataRequest.

In short:
- build a `dataProvider` object with your own contact details
- collect the data and pump it in an array of `guests`
- build a `guestList` object with `start` and `end` from [createDataRequest](#42-process-iris-client-data-requests)
- build your data submission POST request with method `[hdEndpoint].submitGuestList` and following params:
  - `_client` (optional) with your app's CN from [your signing request](#11-generate-certificate-signing-request)
  - `dataAuthorizationToken` taken from [createDataRequest](#42-process-iris-client-data-requests)
  - `guestList` (be aware that `additionalInformation` is required and must at least be an empty string)
- POST your request to your local EPS

### 4.3.3 Send data from app in browser

The data is sent to the proxyEndpoint from createDataRequest via JSON-RPC, in detail: `params.dataRequest.proxyEndpoint`. The method name used is `submitGuestList`. **Do not include [hdEnpoint] in method name as when sending to a local EPS.**

The URL for the submission is build as `https://`[proxyEndpoint]`:32325/data-submission-rpc`.

In short:
- build a `dataProvider` object with your own contact details
- collect the data and pump it in an array of `guests`
- build a `guestList` object with `start` and `end` from [createDataRequest](#42-process-iris-client-data-requests)
- build your data submission POST request with method `submitGuestList` and following params:
  - `_client` (required) with your app's CN from [your signing request](#11-generate-certificate-signing-request)
  - `dataAuthorizationToken` taken from [createDataRequest](#42-process-iris-client-data-requests)
  - `guestList` (be aware that `additionalInformation` is required and must at least be an empty string)
- POST your request to `https://`[proxyEndpoint]`:32325/data-submission-rpc`

**Please consider that health departments will use self-signed certificates on test environment.** 

## 5 Test your implementation

To test your implementation, visit https://client.test.iris-gateway.de 

You can find the password and access data in the slack channel.

There you should find your pushed locations in the search when you start a new event tracking. If you send the request, you should receive a data request. 

## Changelog

### [0.0.10] - 2021-07-02

#### Changed
- New EPS Version. Please have a look especially at the configuration, docker-image, browser submission ports

### [0.0.9] - 2021-06-16

#### Changed
- Corrected sending data submissions from browser
- Added time format for date/time fields in data submissions

### [0.0.8] - 2021-06-16

#### Changed
- Latest eps image version
- Corrected boolean expression
- Additional hint for URL of local EPS

### [0.0.7] - 2021-06-14

#### Added
- Data submission from Browser client to health departments Proxy-Endpoint
- New eps image version 

### [0.0.6] - 2021-06-11

#### Added
- Overview of steps
- Direct links to the sections for each important step

#### Changed
- Step-by-step guide for copy&paste of commandline deployment of local eps
- Some hints for postLocationToSearchIndex
- New eps image version

### [0.0.5] - 2021-06-02

#### Changed
- Submit data when accessible in backend
- New eps image version 

### [0.0.4] - 2021-05-20

#### Added
- Submit data when accessible in backend

#### Changed
- Docker command with new eps version

### [0.0.3] - 2021-05-20

#### Added
- Receiving data requests
- Improved setup instructions
- Test your implementation

### [0.0.2] - 2021-05-08

#### Added
- Overview drawing including steps and descriptions.
- Structure for data requests and data submissions.

### [0.0.1] - 2021-05-04

#### Added
- Initial version of the onboarding document
- Adding locations to search index 
