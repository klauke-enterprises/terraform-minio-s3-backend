# Terraform Minio s3 Backend

Especially when working in a team it's very important to keep track of your terraform state. Instead of
keeping your state in version control (which doesn't make any sense) you may consider keeping your
state in a terraform remote backend. 

You can find the different supported terraform backend in the [terraform docs](https://www.terraform.io/docs/backends/types/index.html).

## The Problem
In this scenario we will use an s3 compatible backend provided by minio. The default way of the s3 terraform backend
is designed for aws. If you're a company you may have compliance requirements like keeping all your 
data in house. 

This is where backends like the [http backend](https://www.terraform.io/docs/backends/types/http.html)
can be used. 

## The Solution
As I'm a fan of s3 storage I wanted to store my terraform state in an s3 storage. To avoid paying 
for cloud storage we will use minio to provide a locally stored s3 storage. 

### Deploy minio via docker-compose

The deployment of a local minio is pretty much straight forward: 
```yaml
version: "3.7"

services:
  
  minio:
    container_name: minio
    restart: always
    image: minio/minio
    volumes:
      - minio-data:/export
    ports:
      - "9000:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: miniominio
    command: server /export

volumes:
  minio-data: {}
```

### Creating the Terraform Bucket

The minio/mc image provides some utilities to help us creating a bucket on startup directly in the 
docker compose. You may not need this in your production setup.

It's sufficient to add the following container:
```yaml
  createbuckets:
    image: minio/mc
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      /usr/bin/mc config host add myminio http://minio:9000 minio miniominio;
      /usr/bin/mc rm -r --force myminio/terraform;
      /usr/bin/mc mb myminio/terraform;
      /usr/bin/mc policy download myminio/terraform;
      exit 0;
      "
```

Let's continue configuring this bucket in terraform.

### Configure Terraform Backend

Let's configure terraform to use our terraform backend:

```hcl-terraform
terraform {
  backend "s3" {
    endpoint                    = "http://localhost:9000"
    bucket                      = "terraform"
    key                         = "terraform.tfstate"
    region                      = "main"
    access_key                  = "minio"
    secret_key                  = "miniominio"
    force_path_style            = true
    skip_requesting_account_id  = true
    skip_credentials_validation = true
    skip_get_ec2_platforms      = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
  }
}
```

Take a special look at the flags below to keys. Ensure to specify `force_path_style = true` to
prevent terraform from resolving the s3 bucket via subdomain in AWS style. We will need
the other flags for compatibility reasons.

## Further information

### Secure your minio

In a production setup you should use minio via HTTPS. I recommend [Traefik](https://docs.traefik.io/) 
as a reverse proxy for automated Let's encrypt certificates.

You can find a demo in [docker-compose-traefik.yml](docker-compose-traefik.yml).

## Special Thanks

To create the initial bucket we make use of the minio/mc image. I got inspired by [Hannes Pries](https://www.hannespries.de/idx-blog-minio--docker-compose.html?page=Blogs&sub=viewBlog&blogId=532) at this point.
