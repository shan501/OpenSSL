# OpenSSL
A simple guide to OpenSSL.
OpenSSL allows you to create SSL certificates within your network without a certfied Certificate Authority.
I used a tutorial made by theurbanpenguin to make this guide.This guide is bassically a summary of his tutorial.If anything is unclear here please check out his amazing video.https://www.youtube.com/watch?v=d8OpUcHzTeg&t=22s

## Basic Introduction
Certificates are used for encrypted communications between devices.SSL are commonly used for encrypting communications on the web but it can also be used with LAN.Many organizations would use a self signed Certifcate Authority to reduce overhead and increase efficency over using a certified third-party Certificate Authority.In this tutorial i will be creating a root Certificate Authority , an Intermediate Certificate Authority , and Servers.

## Create directories
Create the structure for your Certificates Authorities. In this tutorial we will be doing everything on the same system.Typically you will have the root certificate in a seperate offline system for sercurity reasons 

```
mkdir -p ca/{root_ca,sub_ca,server}/{private,certs,newcerts,crl,csr}

```
We just created three directories.The root_ca , sub_ca, and server . Within each directory we created sub-directories private,certs,newcerts,crl,csr

Each of the private directory needs to be secret and shouldnt be access by others so we set the permissions of it to RWX for owner only 

```
chmod -v 700 ca/{root_ca,sub_ca,server}/private

```
