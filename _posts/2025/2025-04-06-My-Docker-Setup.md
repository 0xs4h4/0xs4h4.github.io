---
title: My Docker Setup
date: 2025-04-06 00:00:00 +0800
categories: [Installation]
tags: [docker]
render_with_liquid: false
description: A simple and repeatable way to build your personal cybersecurity tools environment using Docker.
---

## **Step 1: Install Docker**
Run the following commands one by one:

```bash
# Update the system
sudo apt update

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg

# Add Docker’s repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update packages again
sudo apt update

# Install Docker
sudo apt install -y docker-ce

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

```

## **Test Docker**

```bash
sudo docker run hello-world
```

## **Step 3: Build the Docker Image**

In the same folder as your Dockerfile, run:

```bash
sudo docker build -t my-sec-env .
```
This will create a Docker image named my-sec-env with all your tools pre-installed.

## **Step 4: Run Your Security Tools Container**

Start your container:

```bash
sudo docker run -it my-sec-env

```
You’re now inside your own security lab! Use tools like:

nmap
pwntools
curl, ping, htop, and more!

To exit, just type:
```bash
exit
```