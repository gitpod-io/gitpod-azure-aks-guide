# Running Gitpod in [Azure AKS](https://azure.microsoft.com/en-gb/services/kubernetes-service/)

> **IMPORTANT** This guide exists as a simple and reliable way of creating an environment in AKS that can run Gitpod. It is not designed to cater for every situation. If you find that it does not meet your exact needs,
> please fork this guide and amend it to your own needs.

Before starting the installation process, you need:

- An Azure account
  - [Create one now by clicking here](https://azure.microsoft.com/en-gb/free/)
- A user account with "Owner" IAM rights on the subscription
- A `.env` file with basic details about the environment.
  - We provide an example of such file [here](.env.example).
- [Docker](https://docs.docker.com/engine/install/) installed on your machine, or better, a Gitpod workspace :) 

## Azure authentication

For simplicity, this guide does **not** use an Azure [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal).
Authentication is done via an interactive URL, similar to this:

```shell
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code ABC123DEF to authenticate.
```

**To start the installation, execute:**

```shell
make install
```

The whole process takes around twenty minutes. In the end, the following resources are created. These are the Azure versions of the [components Gitpod requires](https://www.gitpod.io/docs/self-hosted/latest/required-components):

- an AKS cluster running Kubernetes v1.21.
- Azure load balancer.
- Azure MySQL database.
- Azure Blob Storage.
- Azure DNS zone.
- Azure container registry.
- [calico](https://docs.projectcalico.org) as CNI and NetworkPolicy implementation.
- [cert-manager](https://cert-manager.io/) for self-signed SSL certificates.

Upon completion, it will print the config for the resources created (including passwords) and instructions on what to do next. **IMPORTANT** - running the `make install` command after the initial install will change 
your database password which will require you to update your KOTS configuration.

## DNS records

> This setup will work even if the parent domain is not owned by a DNS zone in the Azure portal.

The recommended setup is to have `SETUP_MANAGED_DNS` be `true` which will create an
[Azure DNS zone](https://docs.microsoft.com/en-us/azure/dns/dns-zones-records) for your
domain. When the zone is created, you will see various nameserver records (with type `NS`), such
as `ns1-xx.azure-dns.com`, `ns2-xx.azure-dns.net`, `ns3-xx.azure-dns.org` and `ns4-xx.azure-dns.info`
(where `xx` is the number randomly assigned by Azure).

In the DNS manager for the parent domain (eg, `example.com`), create a nameserver record for
each of the nameservers generated by Azure under the subdomain used (eg, `gitpod.example.com`).
This is what it would look like if your parent domain was using Cloudflare.

![Cloudflare DNS manager](./images/dns-record.png "Cloudflare DNS manager")

Once applied, please allow a few minutes for DNS propagation.

### Common errors running make install

- Insufficient regional quota to satisfy request

  Depending on the size of the configured `disks size` and `machine-type`,
  it may be necessary to request an [increase in the service quota](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits)

  *After increasing the quota, retry the installation running `make install`*

- Some pods never start (`Init` state)

  ```shell
  kubectl get pods -l component=proxy
  NAME                     READY   STATUS    RESTARTS   AGE
  proxy-5998488f4c-t8vkh   0/1     Init 0/1  0          5m
  ```

  The most likely reason is that the [DNS01 challenge](https://cert-manager.io/docs/configuration/acme/dns01/) has yet to resolve. If using `SETUP_MANAGED_DNS`, you will need to update your DNS records to point to the Azure DNS zone nameserver.

  Once the DNS record has been updated, you will need to delete all cert-manager pods to retrigger the certificate request

  ```shell
  kubectl delete pods -n cert-manager --all
  ```

  After a few minutes, you should see the `https-certificate` become ready.

  ```shell
  kubectl get certificate
  NAME                        READY   SECRET                      AGE
  https-certificates          True    https-certificates          5m
  ```

## Destroy the cluster and Azure resources

Remove the Azure cluster running:

```shell
make uninstall
```

> The command asks for confirmation:
> `Are you sure you want to delete: Gitpod (y/n)?`

This will destroy the Kubernetes cluster and allow you to manually delete the cloud storage.
