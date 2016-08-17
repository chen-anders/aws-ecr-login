# AWS ECR Login

This script and container aid in loging into AWS EC2 Container Registries (ECR) and updating the docker.tar.gz files on a host (for Docker 1.6+)

## Script Usage
```
./ecr-login.sh [options]
    -r|--region        If not provided, the region will be looked up via AWS metadata (optional on EC2-only).
    -g|--registries    The AWS account IDs to use for the login. Space separated. (Ex: "123456789101, 98765432101")
    -f|--file-location Where the docker.tar.gz should be saved.
    -i|--interval      How often to loop and refresh credentials (optional - default is 21600 - 6 hours).
```

The script can be run from within a Docker container or stand-alone on either AWS infrastructure or traditional servers. When run in a container, you must mount the host path where you wish the `docker.tar.gz` file to be written.

### Docker Container
```
$ docker pull adobeplatform/aws-ecr-login
$ docker run --name aws-ecr-login -v /home/core:/home/core:rw -e "REGION=us-east-1" -e "REGISTRIES=12345678901" -e "FILE_LOCATION=/home/core/docker.tar.gz" -e "INTERVAL=21600"
```

## Usage on Marathon 
To run this on Marathon 1.7/1.8 or newer, here's a sample JSON: 

(Following example is derived from running Marathon on CoreOS)

```
{
  "volumes": null,
  "id": "/ecr-login",
  "cmd": null,
  "args": null,
  "user": null,
  "env": {
    "REGISTRIES": "<ACCOUNT NUMBER (e.g. 000123456789 from ECR repo)>",
    "FILE_LOCATION": "/home/core/docker.tar.gz",
    "AWS_ACCESS_KEY_ID": "<FILL THIS IN>",
    "REGION": "<FILL THIS IN>",
    "AWS_SECRET_ACCESS_KEY": "<FILL THIS IN>",
    "AWS_DEFAULT_REGION": "us-east-1",
    "INTERVAL": "21600"
  },
  "instances": 4,
  "cpus": 0.01,
  "mem": 64,
  "disk": 0,
  "gpus": 0,
  "executor": null,
  "constraints": [
    [
      "hostname",
      "UNIQUE"
    ],
    [
      "hostname",
      "GROUP_BY"
    ]
  ],
  "fetch": null,
  "storeUrls": null,
  "backoffSeconds": 1,
  "backoffFactor": 1.15,
  "maxLaunchDelaySeconds": 3600,
  "container": {
    "docker": {
      "image": "wistiaanders/aws-ecr-login:latest",
      "forcePullImage": true,
      "privileged": false,
      "network": "HOST"
    },
    "type": "DOCKER",
    "volumes": [
      {
        "containerPath": "/home/core",
        "hostPath": "/home/core",
        "mode": "RW"
      }
    ]
  },
  "healthChecks": [
    {
      "protocol": "COMMAND",
      "command": {
        "value": "if [[ $(expr $(date +%s) - $(date +%s -r /home/core/docker.tar.gz)) -gt 21600 ]]; then exit 1; else exit 0; fi"
      },
      "gracePeriodSeconds": 60,
      "intervalSeconds": 60,
      "timeoutSeconds": 20,
      "maxConsecutiveFailures": 2,
      "portType": "PORT_INDEX"
    }
  ],
  "readinessChecks": null,
  "dependencies": null,
  "upgradeStrategy": {
    "minimumHealthCapacity": 1,
    "maximumOverCapacity": 1
  },
  "labels": {},
  "acceptedResourceRoles": [
    "slave_public",
    "*"
  ],
  "ipAddress": null,
  "residency": null,
  "secrets": null,
  "taskKillGracePeriodSeconds": null,
  "portDefinitions": [
    {
      "protocol": "tcp",
      "port": 10000
    }
  ],
  "requirePorts": false
}
```


### Usage on AWS Infrastructure
Running on AWS means that the region can be determined automatically via the AWS metadata service and does not need to be provided via the command line.

### Usage on Traditional Servers
Running on traditional servers means that the region must be provided. Failure to provide the region will result in the script hanging as it attempts to contact the (unavailable) AWS metadata service.

```
$ ./ecr-login.sh --region us-east-1 --registries 12345678901 --file-location /home/core/docker.tar.gz --interval 21600
```
