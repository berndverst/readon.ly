---
date: 2017-04-05T18:24:08-07:00
subtitle: Using Azure Storage and Azure CDN for cheap static website hosting with free SSL
tags: [azure, cdn, storage, custom-domain]
title: Static websites on Azure
---
I'm a big fan of generating static websites from templates. This blog uses the excellent [Hugo](https://gohugo.io/) static site generator. The big question of course is how to host your static website in an easy and cost-effective manner.
<!--more-->

In this tutorial I will be showing you how to use Azure Storage and Azure CDN to easily host your static website for mere pennies a month.

### Hosting a static website from a blob storage container

#### A few prerequisites:

- Use BASH, ZSH or other shells with BASH-compatibility
- Make sure to install the [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- Your static website content lives in the folder `public` within the current directory.


```bash
# Creates a resource group
az group create --location "West US" \
--name berndverst-staticwebsite-group
```


```bash
# Creates a blob storage account
#
# Locally-redundant storage (LRS) should suffice.
# We will require frequent access and therefore choose the Hot tier.

az storage account create --location "West US" \
--name berndverststaticwebsite \
--resource-group berndverst-staticwebsite-group \
--kind BlobStorage --sku Standard_LRS --access-tier Hot
```


```bash
# Exports our storage account's connection string to variable $CON
#
# We will frequently need to provide the connection string.

export CON=$(az storage account show-connection-string \
--name berndverststaticwebsite \
--resource-group berndverst-staticwebsite-group | \
grep -Eo "\"DefaultEndpointsProtocol.*")
```


```bash
# Creates a container or bucket for our static website
az storage container create --name blog --connection-string $CON
```


```bash
# Makes the container publicly (anonymously) accessible
az storage container set-permission --connection-string $CON \
--name blog --public-access container
```


```bash
# Uploads a single file to our container
az storage blob upload --container-name blog \
--connection-string $CON --name index.html \
--file public/index.html
```


Thie following doesn't work -- it sets the wrong content type:
```bash
# We can upload files of the same content-type in bulk.
# This would set a content-type of application/octet-stream for all files.

az storage blob upload-batch --type block \
--connection-string $CON --destination "blog" --source public/
```

Instead we have to do this:
```bash
# Upload all files of a specific content type in batches.

# Copy all CSS files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "text/css" --pattern "*.css"

# Copy all PNG images
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "image/png" --pattern "*.png"

# Copy all GIF images
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "image/gif" --pattern "*.gif"

# Copy all JPG images
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "image/jpg" --pattern "*.jpg"

# Copy all HTML files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "text/html" --pattern "*.html"

# Copy all JavaScript files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "application/javascript" --pattern "*.js"

# Copy all XML files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "text/xml" --pattern "*.xml"

# Copy all Icon files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "image/x-icon" --pattern "*.ico"

# Copy all ZIP files
az storage blob upload-batch --connection-string $CON \
--destination "blog" --source public/ \
--content-type "application/zip" --pattern "*.zip"

# ... Continue doing this for other file types
```

### CDN, Custom Domain Mapping & SSL
Unfortunately, this is where our CLI support ends, so we'll be heading over to the **Azure portal**.

Search for `CDN` and create a new `Standard Verizon` CDN (this is the only CDN supporting custom domains with SSL).

![Creating a CDN](/img/azure-static-website/cdn.png)

Next we create a CDN Endpoint (or distribution). This maps a CDN URL to our blob storage container origin.

![Creating a CDN Endpoint](/img/azure-static-website/endpoint.png)

When you have completed the set up things should look like this:

![CDN Endpoint Summary](/img/azure-static-website/endpoint-overview.png)

Now let's add a custom domain.

![Creating a custom domain mapping](/img/azure-static-website/custom-domain.png)

Before we can configure our custom domain mapping we need to create a `CNAME` entry for our desired custom domain (in this case `blog.demo.readon.ly`) pointing to our CDN endpoint URL (here `staticblog.azureedge.net`) in our DNS.

Confirming the DNS record has been set up correctly:
```bash
dig cname blog.demo.readon.ly +short
# returns "staticblog.azureedge.net."
```

Click on your newly created custom domain and enable SSL.

![Enable SSL for custom domain](/img/azure-static-website/custom-domain-ssl.png)

Confirm ownership of your domain by clicking the link in the *digicert* verification email.

![Confirm domain ownership for SSL cert issuance](/img/azure-static-website/cert.png)

SSL support should start within the next 8 hours.

**And we are done.** `blog.demo.readon.ly` now serves the content of our blob storage account.

#### Caveats

Azure Blob Storage does not yet support default documents for folders, so requests to `/` or `/folder` won't load `/index.html` or `/folder/index.html` respectively

- You will need to use full file names for links.
- Visitors to your site must visit customdomain.com/index.html.



### Cost

If you are wondering how much all of this would actually cost us. See below the calculations for my little blog.

Service type | Custom name | Region | Description | Estimated Cost
--- | --- | --- | --- | ---
Storage | Storage | West US | 1/GB storage: block type, Basic tier, LRS redundancy, hot access tier. , 5000 x10,000 put/create container transactions , 1000 x10,000 others transactions (except delete which is free), 1/GB data retrieval, 1/GB data write, 1/GB data geo-replication. | $0.05
Content Delivery Network | Content Delivery Network | | 1 GB/Month Zone1, 0 GB/Month Zone 2, 0 GB/Month Zone 3, 0 GB/Month Zone 4, 0 GB/Month Zone 5 | $0.09
Data Transfers | Bandwidth | West US | 5 GB/Month Zone 1 (North America, Europe) | $0.00
Support | | | Support | $0.00
 | | | Monthly Total | $0.14
 | | | **Annual Total** | **$1.63**
				
*Disclaimer*:
All prices shown are in US Dollar ($). This is a summary estimate, not a quote. For up to date pricing information please visit https://azure.microsoft.com/pricing/calculator/
This estimate was created at 4/5/2017 10:58:48 PM UTC.				
