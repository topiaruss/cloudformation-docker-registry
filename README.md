CloudFormation template for an S3-backed private [Docker Registry](https://github.com/dotcloud/docker-registry) with password protection and HTTPS.

Prerequisites:
* SSL certificate (if it's untrusted, you'll need to add it to the system keystore or pass use Docker's `--insecure` option)
* (Optional) Route 53 hosted zone for the desired registry address (e.g., `mycompany.com` for `docker.mycompany.com`)

## Overview

This template bootstraps a private Docker Registry.

Registry servers are launched in an auto scaling group using public AMIs running Ubuntu 14.04 LTS and pre-reloaded with Docker and Runit.  If you wish to use your own image, simply modify `RegionMap` in the template file.

The servers register with an Elastic Load Balancer, an Alias for which is created in Route 53 (if you pass `DnsZone` and `DnsPrefix`). When tagging repositories, use this (or a higher-level record) instead of the ELB's raw DNS record as your registry address instead (e.g., `docker.mycompany.com/myimage` instead of `mystack-do-ElasticL-AJK...elb.amazonaws.com/myimage`).

The ELB listens on HTTPS using the specified SSL cert. Each registry server is password-protected by an nginx proxy. The ELB performs health checks against `/v1/_ping`. If an instance fails these health checks for several minutes, the ASG will terminate the instance and bring up a replacement.

The registry is run via a Docker image specified as a Parameter. You can use the default or provide your own.

To adjust registry capacity, simply increment or decrement the auto scaling group. Node addition/removal should be handled transparently by the ASG and ELB.

Note that this template must be used with Amazon VPC. New AWS accounts automatically use VPC, but if you have an old account and are still using EC2-Classic, you'll need to modify this template or make the switch.

## Usage

### 1. Clone the repository
```bash
git clone https://github.com/mbabineau/cloudformation-docker-registry.git
```

### 2. Create an Admin security group
This is a VPC security group containing access rules for cluster administration, and should be locked down to your IP range, a bastion host, or similar. This security group will be associated with the Registry servers.

Inbound rules are at your discretion, but you may want to include access to:
* `22 [tcp]` - SSH port
* `80 [tcp]` - Private nginx proxy port (troubleshooting only)
* `5000 [tcp]` - Private registry port (troubleshooting only)

### 3. Upload an SSL certificate
Upload an SSL certificate to IAM ([instructions](http://docs.aws.amazon.com/IAM/latest/UserGuide/InstallCert.html)). This certificate must be valid for the registry DNS address and be trusted by the clients (if self-signed, you must add it to the clients' keystores or use Docker's `--insecure` flag).

Once uploaded, you'll need the ARN for this certificate. You can do this via [aws-cli](https://github.com/aws/aws-cli):
```console
$ aws iam list-server-certificates
{
    "ServerCertificateMetadataList": [
        {
            "Path": "/", 
            "Arn": "arn:aws:iam::123456789:server-certificate/docker.mycompany.com", 
            "ServerCertificateId": "AAAAAIZG21OOT4VVVVV", 
            "ServerCertificateName": "docker.mycompany.com", 
            "UploadDate": "2014-06-05T20:06:43Z"
        }
    ]
}
```

### 4. Generate the auth strings
Docker will be password-protected using an htpasswd file. The `RegistryAuth` parameter takes a comma-delimited list of these auth strings. You can generate an auth string using htpasswd:
```console
$ htpasswd -bn admin docker
admin:$apr1$SteyXrya$20Smsae8n4EkGTX2u17ou0
```

### 5. Launch the stack
Launch the stack via the AWS console, a script, or `aws-cli`.

See `docker-registry.json` for the full list of parameters, descriptions, and default values.

Interface using `aws-cli`:
```bash
aws cloudformation create-stack \
    --template-body file://docker-registry.json \
    --stack-name <stack> \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=AvailabilityZones,ParameterValue=<availability-zone> \
        ParameterKey=Ami,ParameterValue=<ami-id-for-your-region> \
        ParameterKey=KeyName,ParameterValue=<key> \
        ParameterKey=S3Region,ParameterValue=<region> \
        ParameterKey=RegistryAuth,ParameterValue='<auth_string_1> <auth_string_2>' \
        ParameterKey=S3Bucket,ParameterValue=<bucket> \
        ParameterKey=SslCertificate,ParameterValue=<cert_arn> \
        ParameterKey=DnsPrefix,ParameterValue=<subdomain> \
        ParameterKey=DnsZone,ParameterValue=<dns_zone> \
        ParameterKey=VpcId,ParameterValue=<vpc_id> \
        ParameterKey=Subnets,ParameterValue='<subnet_id_1> <subnet_id_2>' \
        ParameterKey=AdminSecurityGroup,ParameterValue=<sg_id>
```

Here are some example values:
```bash
aws cloudformation create-stack \
    --template-body file://docker-registry.json \
    --stack-name docker-registry \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=AvailabilityZones,ParameterValue=eu-west-1c \
        ParameterKey=Ami,ParameterValue=ami-f3561384 \
        ParameterKey=KeyName,ParameterValue=MyDockerRegistryKeypair \
        ParameterKey=S3Region,ParameterValue=eu-west-1 \
        ParameterKey=RegistryAuth,ParameterValue='admin:$apr1$Xm1YEZND$3OsrpDnlDdld8TAEwItOj0' \
        ParameterKey=S3Bucket,ParameterValue=my-docker-registry-bucket \
        ParameterKey=SslCertificate,ParameterValue=arn:aws:iam::377312681624:server-certificate/dockreg_myorg_com_certificate \
        ParameterKey=DnsPrefix,ParameterValue=dockreg \
        ParameterKey=DnsZone,ParameterValue=myorg.com \
        ParameterKey=VpcId,ParameterValue=vpc-2324B043 \
        ParameterKey=Subnets,ParameterValue='subnet-1a27c541' \
        ParameterKey=AdminSecurityGroup,ParameterValue=sg-31c5ff51
```

It's essential that the availability zone of the the Subnet matches the AvailabilityZones value.

### 6. Test your registry
Once the stack has been provisioned, try calling the registry using credentials associated with one of the auth strings. A verbose curl should give HTTP/1/1 200 OK. If you receive a 501 code, try again in a few minutes -- your server may still be initializing:
```console
$ curl -vv -u <user>:<password> https://dockreg.myorg.com
* Rebuilt URL to: https://dockreg.myorg.com/
* Hostname was NOT found in DNS cache
*   Trying 54.264.180.59...
* Connected to dockreg.topia.com (54.264.180.59) port 443 (#0)
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* Server certificate: dockreg.myorg.com
* Server certificate: COMODO RSA Domain Validation Secure Server CA
* Server certificate: COMODO RSA Certification Authority
* Server certificate: AddTrust External CA Root
* Server auth using Basic with user 'admin'
> GET / HTTP/1.1
> Authorization: Basic YWRtaW42ZG9ja2Vy
> User-Agent: curl/7.37.1
> Host: dockreg.myorg.com
> Accept: */*
>
< HTTP/1.1 200 OK
< Cache-Control: no-cache
< Content-Type: application/json
< Date: Wed, 24 Jun 2015 10:12:35 GMT
< Expires: -1
< Pragma: no-cache
* Server nginx/1.6.2 (Ubuntu) is not blacklisted
< Server: nginx/1.6.2 (Ubuntu)
< Content-Length: 28
< Connection: keep-alive
<
* Connection #0 to host dockreg.myorg.com left intact
```

_Note: if you didn't pass `DnsZone` and `DnsPrefix`, you'll want to set up a CNAME or Alias for the created ELB_

To use the new registry, just generate a new `~/.dockercfg` by hand ([details](http://docs.docker.io/use/workingwithrepository/#authentication-file)) or via `docker login`:
```console
$ docker login docker.mycompany.com
Username: admin
Password: 
Email: user@mycompany.com
Login Succeeded
$ cat ~/.dockercfg
{"docker.mycompany.com":{"auth":"YWRtaW46ZG9ja2Vy","email":"user@mycompany.com"}}
```

You can now tag and push images to your new registry:
```console
$ docker tag 31bd83ec56f3 docker.mycompany.com/myimage
$ docker push docker.mycompany.com/myimage
The push refers to a repository [docker.mycompany.com/myimage] (len: 1)
Sending image list
Pushing repository docker.mycompany.com/myimage (1 tags)
Image 511136ea3c5a already pushed, skipping
9e0876577d2f: Image successfully pushed 
60d8d9423165: Image successfully pushed 
31bd83ec56f3: Image successfully pushed 
Pushing tag for rev [31bd83ec56f3] on {https://docker.mycompany.com/v1/repositories/myimage/tags/latest}
```

###7. Possible server filesystem full on current registry

As of writing, the container starting the current version of registry leaves the EC2 instance disk full. The symptom
is a push that starts running then fails with a server error. The quick answer is to 
([clean up the instance docker volume](https://registry.hub.docker.com/u/martin/docker-cleanup-volumes/)). 

SSH to the instance and:
```bash
sudo docker run -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker:/var/lib/docker \
  --rm martin/docker-cleanup-volumes
```
