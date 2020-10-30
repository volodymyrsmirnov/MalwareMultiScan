# MalwareMultiScan

![Scanners](https://github.com/mindcollapse/MalwareMultiScan/workflows/Scanners/badge.svg) ![API](https://github.com/mindcollapse/MalwareMultiScan/workflows/API/badge.svg) ![UI](https://github.com/mindcollapse/MalwareMultiScan/workflows/UI/badge.svg) ![Tests](https://github.com/mindcollapse/MalwareMultiScan/workflows/Tests/badge.svg?event=push)

Self-hosted [VirusTotal](https://www.virustotal.com/) wannabe API for scanning URLs and files by multiple antivirus solutions.

![MalwareMultiScan UI](.github/img/malware-multi-scan-ui.gif)

## Introduction

I faced a need to scan user-uploaded files in one of my work projects in an automated mode to ensure they don't contain any malware. Using VirusTotal was not an option because of a) legal restrictions and data residency limitations b) scanning by hash-sums would not be sufficient because the majority of files are generated / modified by users. 

After googling, I stumbled upon a fantastic [maliceio/malice](https://github.com/maliceio/malice) project. Unfortunately, it [looks abandoned](https://github.com/maliceio/malice/issues/100#issuecomment-529246119), and most plugins do not work for the moment. In addition to that, I had an intention to use the .NET stack to align with the internal infrastructure.

In the end, it's nothing but the set of Docker containers running the agent. That agent downloads the remote file to the temp folder, then launches the vendor command-line scanning utility with proper arguments, and parses the output with a regular expression to extract a detected malware name.

## Installation & Usage

**IMPORTANT**: MalwareMultiScan is not intended as a publicly-facing API / UI. It has (intentionally) no authorization, authentication, rate-limiting, or logging. Therefore, it should be used only as an internal / private API or behind the restrictive API gateway.

Whole solution can be started with `docker-compose up` executed in a root folder of repository.

It can be also deployed to the Docker Swarm cluster by using the command `docker stack deploy malware-multi-scan --compose-file docker-compose.yaml`.

After the start the Demo Web UI will become available under http://localhost:8888.

See [components](#components) chapter below and the [docker-compose.yaml](docker-compose.yaml) file.

### Configuration

Configuration of API and Scanners is performed by passing the environment variables. Descriptions and default values are provided below.

#### MalwareMultiScan.Api

* `ConnectionStrings__Mongo=mongodb://localhost:27017` - MongoDB connection string.

* `DatabaseName=MalwareMultiScan` - MongoDB collection name.

* `ConnectionStrings__RabbitMQ=host=localhost` - RabbitMQ connection string. See [EasyNetQ Wiki](https://github.com/EasyNetQ/EasyNetQ/wiki/Connecting-to-RabbitMQ) for details.

* `ResultsSubscriptionId=mms.results` - RabbitMQ subscription id for scan results. Should match with values defined for scanners.

* `MaxFileSize=52428800` - Maximum size of a file that can be handled for the file scanning. The size of the URL content is not verified.

* `BackendsConfiguration=backends.yaml` - Path to the [backends.yaml](MalwareMultiScan.Api/backends.yaml) file.

#### MalwareMultiScan.Scanner

* `ConnectionStrings__RabbitMQ=host=localhost` - RabbitMQ connection string. See [EasyNetQ Wiki](https://github.com/EasyNetQ/EasyNetQ/wiki/Connecting-to-RabbitMQ) for details.

* `ResultsSubscriptionId=mms.results` - RabbitMQ subscription id for scan results. Should match with values defined for scanners.

* `BackendType=Dummy` - A type of scanner backend used by the running instance. Should correspond to the [BackendType](MalwareMultiScan.Backends/Enums/BackendType.cs) enum.

* `MaxScanningTime=60` - Scan time limit. It is used not just for actual scanning but also for getting the file.

* `WorkerCount=4` - Number of workers for parallel scanning.

#### MalwareMultiScan.Ui

* `API_URL=http://localhost:5000` - Absolute URL incl. port number for the running instance of MalwareMultiScan.Api.  

### API Endpoints

* POST `/api/queue/url` with a `url` parameter passed via the form data.. Returns `201 Accepted` response with a [ScanResult](MalwareMultiScan.Api/Data/Models/ScanResult.cs) or `400 Bad Request` error.

* POST `/api/queue/file` with a `file` parameter passed via the form data. Returns `201 Accepted` response with a [ScanResult](MalwareMultiScan.Api/Data/Models/ScanResult.cs) or `400 Bad Request` error.

* GET `/api/results/{result-id}` where `{result-id}` corresponds to the id value of a [ScanResult](MalwareMultiScan.Api/Data/Models/ScanResult.cs). Returns `200 OK` response with a [ScanResult](MalwareMultiScan.Api/Data/Models/ScanResult.cs) or `404 Not Found` error.

Both `/api/queue/url` and `/api/queue/file` also accept an optional `callbackUrl` parameter with the http(s) URL in it. This URL will be requested by the POST method with JSON serialized [ScanResult](MalwareMultiScan.Api/Data/Models/ScanResult.cs) in a body on every update from scan backends.

## Supported Scan Engines

| Name | Dockerfile | Enabled | Comments |
|------|------------|---------|----------|
| [ClamAV](https://www.clamav.net/) | [Clamav.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/Clamav.Dockerfile) | :white_check_mark: | |
| [Comodo](https://www.comodo.com/home/internet-security/antivirus-for-linux.php) | [Comodo.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/Comodo.Dockerfile) | :white_large_square: | |
| [DrWeb](https://download.drweb.com/linux/?lng=en) | [DrWeb.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/DrWeb.Dockerfile) | :white_large_square: | Pass license key to the DRWEB_KEY build arg. |
| Dummy | [Dockerfile](MalwareMultiScan.Scanner/Dockerfile) | :white_check_mark: | Scan backend made for testing. Returns Malware.Dummy.Result threat for every scan after 5 seconds. |
| [KES](https://support.kaspersky.com/kes10linux) | [KES.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/KES.Dockerfile) | :white_large_square: | Pass license key to the KES_KEY build arg. KES 11 does not work in Docker. |
| [McAfee](https://www.mcafee.com/enterprise/en-us/products/virusscan-enterprise-for-linux.html) | [McAfee.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/McAfee.Dockerfile) | :white_large_square: | |
| [Sophos](https://www.sophos.com/en-us/support/documentation/sophos-anti-virus-for-linux.aspx) | [Sophos.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/Sophos.Dockerfile) | :white_large_square: | |
| [Defender](https://github.com/taviso/loadlibrary#windows-defender) | [WindowsDefender.Dockerfile](MalwareMultiScan.Backends/Dockerfiles/WindowsDefender.Dockerfile) | :white_check_mark: | |

More scan backends can be added in the future. Some of the popular ones do not have command line scanning utility, Linux version, or don't start in Docker container. Feel free to raise an issue if you know any in addition to the list above.

## Components

### Prerequisites

* **MongoDB** of version 3.x. Used for storing scan results and files in GridFS. The communication is happening through the [official C#/.NET driver](https://docs.mongodb.com/drivers/csharp).

* **RabbitMQ** of version 3.x. Used for IPC and scan tasks queueing. The communication is happening through the [EasyNetQ](https://github.com/EasyNetQ/EasyNetQ) library.

* **Docker** and **docker-compose** running under Windows (in Linux containers mode), Linux, or OSX. Docker Compose is needed only for test / local deployments.

* **Optional**: DockerSwarm / Kubernetes cluster for scaling up the scanning capacities.

### Parts

* [MalwareMultiScan.Api](MalwareMultiScan.Api) - Simple ASP.NET Core WebApi for queueing files & urls for the scan and returning the result. Also acts as a receiver of scan results from the scanning backend nodes. See [Dockerfile](MalwareMultiScan.Api/Dockerfile). Configuration of available backends is performed via the [backends.yaml](MalwareMultiScan.Api/backends.yaml) location passed via the `BackendsConfiguration` environment variable. 

* [MalwareMultiScan.Backends](MalwareMultiScan.Backends) - Shared components between API and Worker. Includes Dockerfiles and implementation classes for third-party vendor scan backends.

* [MalwareMultiScan.Scanner](MalwareMultiScan.Scanner) - .NET Core Worker service subscribes to messages corresponding to the backend id, then fires up scanning command-line utility, and parses the output. See [Dockerfile](MalwareMultiScan.Scanner/Dockerfile). The image of MalwareMultiScan.Scanner acts as a base image for the rest of the scan backends. Check Dockerfiles from the [table above](#supported-scan-engines) for details.

* [MalwareMultiScan.Ui](MalwareMultiScan.Ui) - Nuxt.js TypeScript SPA for demoing the API capabilities. See [Dockerfile](MalwareMultiScan.Ui/Dockerfile). 
 
 ## Plans
 
 See [issues](https://github.com/mindcollapse/MalwareMultiScan/issues) for the list of planned features, bug-fixes, and improvements.