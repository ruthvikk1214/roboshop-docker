# Roboshop Docker Project

This repository contains the Dockerized configuration for the **Roboshop** application, a multi-container microservices application.

## Component Architecture

The application consists of the following components:

- **Frontend**: The user interface. Routes requests to `catalogue`, `user`, and other backend services.
- **MongoDB**: Database store for product catalog details.
- **Redis**: In-memory data store for user sessions and cart details.
- **Catalogue**: Microservice handling catalog-related requests (depends on MongoDB).
- **User**: Microservice managing user profiles and logins (depends on Redis).
- **Cart**: Microservice managing user carts (depends on Redis).
- **Shipping**: Microservice handling shipping calculations.

---

## Networking and Inter-Component Communication

> [!IMPORTANT]
> **Custom Bridge Network Requirement**
> To allow secure and seamless communication between the microservices, a **custom bridge network** must be used.
>
> **Why is this necessary?**
> By default, containers on the default bridge network cannot resolve each other by container/service name via DNS. Using a custom bridge network enables Docker's automatic DNS resolution, allowing containers to connect to each other using their container name (e.g., `mongodb`, `redis`) as the hostname.
>
> In the `compose.yaml` configuration, this custom bridge network is defined under the `networks` block:
> ```yaml
> networks:
>   default:
>     name: roboshop
> ```
> 
> If you are running the containers individually (without Docker Compose), you must first create this network manually before launching the containers:
> ```bash
> docker network create --driver bridge roboshop
> ```
> And connect each container to it:
> ```bash
> docker run --network roboshop ...
> ```

---

## Getting Started

### Using Docker Compose
To build and start the components together:
```bash
docker compose up -d --build
```
# Increase Disk Space on RHEL 9.7 EC2 (LVM)

> **Note:** While creating the EC2 instance, choose **50 GB** as the root volume size instead of the default size. On RHEL 9.x, the filesystem is **not automatically expanded** to use the entire EBS volume, so the following steps are required.

## Verify the current disk layout

```bash
lsblk
df -h
sudo vgs
sudo lvs
```

## Install `growpart`

```bash
sudo dnf install -y cloud-utils-growpart
```

## Extend the root partition

```bash
sudo growpart /dev/nvme0n1 4
```

> Replace `4` with your root partition number if it is different.

## Resize the LVM Physical Volume

```bash
sudo pvresize /dev/nvme0n1p4
```

## Verify free space in the Volume Group

```bash
sudo vgs
```

`VFree` should now show the newly available space.

## Extend the `/var` Logical Volume

Since Docker stores images and containers under `/var/lib/docker`, it's recommended to extend the `/var` logical volume.

Use all available free space:

```bash
sudo lvextend -r -l +100%FREE /dev/RootVG/varVol
```

Or extend by a specific size (example: 20 GB):

```bash
sudo lvextend -r -L +20G /dev/RootVG/varVol
```

## Verify

```bash
df -h
```

The `/var` filesystem should now reflect the increased size.

---

### Why is this required?

Although the EC2 instance is created with a **50 GB EBS volume**, RHEL 9.x initially creates a smaller LVM partition (around 20 GB by default). The remaining space is left unallocated and must be manually extended before Docker workloads can use it.
