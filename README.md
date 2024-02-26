# Providers in the private registry
This is a test repository to add an existing public provider to the private registry of Terraform Cloud.

The `oci` [public provider Oracle](https://registry.terraform.io/providers/oracle/oci/latest/docs) will be added to the private registry of Terraform Cloud

**NOTE**: The exact same steps can be followed for a Terraform Enterprise installation.

## Instructions

This repository has been created only for learning purposes and it is based on the [official documentation from HashiCorp](https://developer.hashicorp.com/terraform/cloud-docs/registry/publish-providers) and [this Github](https://github.com/slavrdorg/tfc-publish-private-provider/blob/main/README.md) step by step guide.

All the commands have been executed on a **MacOS system**. For other operating systems, please adjust accordingly where needed.

Please also note the following 2 KB articles 
- [https://support.hashicorp.com/hc/en-us/articles/14522974687507-Upload-Provider-to-Private-Registry](https://support.hashicorp.com/hc/en-us/articles/14522974687507-Upload-Provider-to-Private-Registry)
- [https://support.hashicorp.com/hc/en-us/articles/26638397057299-Error-Failed-to-install-provider-authentication-signature-from-unknown-issuer](https://support.hashicorp.com/hc/en-us/articles/26638397057299-Error-Failed-to-install-provider-authentication-signature-from-unknown-issuer)

### Prerequisites

- [X] [Terraform](https://www.terraform.io/downloads)
- [X] [Terraform Cloud account](https://app.terraform.io/public/signup/account)

## How to Use this Repo

- Clone this repository:
```shell
git clone https://github.com/munnep/tfc_custom_provider_private_registry.git
```

- Go to the directory where the repository is stored:
```shell
cd tfc_custom_provider_private_registry
```

## Steps for preparing the new provider to be uploaded to the private registry of Terraform Cloud

- Download the providers zipfiles you need from the github page of the provider. Please download the version for mac darwin and linux. We download both because we need to use the provider on our macbook and on Terraform Enterprise/Cloud and this requires the linux version of the provider
https://github.com/oracle/terraform-provider-oci/releases/tag/v5.30.0

- files should look like the following now

```
terraform-provider-oci_5.30.0_darwin_amd64.zip
terraform-provider-oci_5.30.0_linux_amd64.zip
```
- Install the `jq` package:
```shell
brew install jq
```

- Install the `gnupg` package, needed for creating a gpg public key:
```shell
brew install gnupg
```

- Create a GPG keypair to sign the release using the RSA algorithm:
```shell
 gpg --full-generate-key
```
output
```
gpg (GnuPG) 2.4.3; Copyright (C) 2023 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 
Requested keysize is 3072 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: patrick_oci
Email address: patrick_oci@test_oci.com
Comment: testing
You selected this USER-ID:
    "patrick_oci (testing) <patrick_oci@test_oci.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: revocation certificate stored as '/Users/patrickmunne/.gnupg/openpgp-revocs.d/121AD89CA07D84F400480621CB3AD3052843121E.rev'
public and secret key created and signed.

pub   rsa3072 2024-02-19 [SC]
      121AD89CA07D84F400480621CB3AD3052843121E
uid                      patrick_oci (testing) <patrick_oci@test_oci.com>
sub   rsa3072 2024-02-19 [E]
```

- Export the public gpg key:
```shell
gpg -o gpg-key.pub -a --export <key_id>
```

- Create a file with the shasums for the binaries of the new provider `vra2` and the version 0.7.1:
```shell
shasum -a 256 terraform-provider-oci_5.30.0*.zip > terraform-provider-oci_5.30.SHA256SUMS
```

Note that a file `terraform-provider-oci_5.30.SHA256SUMS` has been created 

- Create a detached signature using a gpg key:
```shell
gpg -k
```
- use the same signing key we used previously
```shell
gpg  --default-key 121AD89CA07D84F400480621CB3AD3052843121E -sb terraform-provider-oci_5.30.SHA256SUMS
```

Note that a file `terraform-provider-oci_5.30.SHA256SUMS.sig` has been created 

## Steps to publish the provider to the private registry of Terraform Cloud

- Export your API token from Terraform Cloud as an environment variable:
```shell
export TOKEN=0zGXzPqxHtbqUg.atlasv1.xxxxxxxx
```

- Create the payload file named `gpg-key-payload.json` for the GPG key and generate your `ascii-armor` value with the following command (where your gpg key is in the file `gpg-key.pub):
```shell
sed 's/$/\\n/g' gpg-key.pub | tr -d '\n\r'
```

- The payload json file `gpg-key-payload.json` should look like this:
```json
{
  "data": {
    "type": "gpg-keys",
    "attributes": {
      "namespace": "<your-org>",
      "ascii-armor": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n\nmI0E...6L\n-----END PGP PUBLIC KEY BLOCK-----\n"
    }   
  }
}
```

- Add the GPG key to the private registry of the Terraform Cloud:
```shell
curl -sS \
    --header "Authorization: Bearer $TOKEN" \
    --header "Content-Type: application/vnd.api+json" \
    --request POST \
    --data @gpg-key-payload.json \
    https://tfe67.aws.munnep.com/api/registry/private/v2/gpg-keys | jq '.'
```
```json
{
  "data": {
    "type": "gpg-keys",
    "id": "4",
    "attributes": {
      "ascii-armor": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n\nmQGNBGXTY24BDADSXDCMGiR80JDrNfEua4/n=wVNT\n-----END PGP PUBLIC KEY BLOCK-----\n",
      "created-at": "2024-02-19T14:27:43Z",
      "key-id": "CB3AD3052843121E",
      "namespace": "test",
      "source": "",
      "source-url": null,
      "trust-signature": "",
      "updated-at": "2024-02-19T14:27:43Z"
    },
    "links": {
      "self": "/v2/gpg-keys/4"
    }
  }
}
```

Save the value of the `key-id`: `CB3AD3052843121E`

- Create the payload file named `provider-payload.json`:
```json
{
    "data": {
      "type": "registry-providers",
        "attributes": {
        "name": "oci",
        "namespace": "test",
        "registry-name": "private"
      }
    }
  }
```

- Create the provider `oci` in the private registry of Terraform Cloud/Enterprise
```shell
curl -sS \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @provider-payload.json \
  https://tfe67.aws.munnep.com/api/v2/organizations/test/registry-providers | jq '.'
  ```

- Create the payload file named `provider-version-payload.json` for the version:
```json
{
  "data": {
    "type": "registry-provider-versions",
    "attributes": {
      "version": "5.30.0",
      "key-id": "<your-gpg-key-id>",
      "protocols": ["5.0"]
    }
  }
}
```

- Create the provider version in the private registry of Terraform Cloud:
```shell
curl -sS \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @provider-version-payload.json \
  https://tfe67.aws.munnep.com/api/v2/organizations/test/registry-providers/private/test/oci/versions | jq '.'
```

- Save your URLs for `shasums-upload` and `shasums-sig-upload`:
```json
      "shasums-upload": "https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6aHE2cHlaZ1BuSXQ3OUpuOU14OUhpcHd1NndBNzBtaGE1SStDRlBqVG1ReXlVdnVNUVppSVk2ZnhjSy8vUVRGMldmcGloakZKZFNuRWthUVpWZSt3WDU0V0dsa0I1cnU1NC9qT2tBZWZNamVWVHEyZ1ZGSFJndkFLS0w5MzlVbEpuVHBDSlA5dDR2alRXektSOWRrcG5Ydlk2SFhJUHNpRVQybW9WZWprYkNwM1c4Y3FMdnpCQnFCektVd1VhMlROVlVMb1pYL2VOQXg4QWFkNTkyNjlwTTUzS0lUR0Vld0Q2eHVMa3ZuTWU4VXNaQ3duT2Zob0IzSzZQaVFvYjN2L2txa200bHdmcjk2T093ei9Xd0JXaWZsWU00MXJCb2RKZ1JLeFJWcktKSGs9",
      "shasums-sig-upload": "https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6VnEvMjU5VEFrRDJReWFReDdHbkJxSzJ0R3JaT3AxWWJZb2Fsa28vY2s2MlFjQ0R3U1ZhRXliNUp4RHFYVTdLdlcya2dXc1IwampWdkZXaXZkQm5OR0dnWThWK0RFYU9EbWNpZjhMN2Q4bEJoRHZYUkVEMjNkNHBCNlV0aU1jU3NPTGs5dDdjOEhmcWJzTFFHYVVITDRJYWFoS3FuWHNzcWVmc3JRZFJuQXkzYWtCQkxvQTVQTzBVbXRGSGtkRkVkQ201TGpick01STQycXhWV3pKbjJMQlZPcnM2Y1NqOFAyV1V6OHN5Vlh3NExpeEJjUDA2WkF5djc1dFJFR24xbFowK0Y2RFhTTDBLb2tnbjJ5Y3NhN2FLTXM5OVZUdEE3aUc4TXFsbU4yVElkYStMYg"
    }
   
```

- Upload the `shasum` file to the URL:
```shell
curl -T terraform-provider-oci_5.30.SHA256SUMS https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6bk1hdVZjRkNJU3hBT1VWR1YxWjJ4ajVEZThqeUh2ZjV1U3RvUnA3b1hXcVlaSjJNT3N1aHZ3bHZxeklYakIzN29QOGtkeXMwR2Z4RHRNWk5RV283bWMvUEkxSkJBbUUwYS9rbFRpL0VWYW9oRlJCaFBCdHhrUGJPbzNBVjBpZWxpV2YxQmE1bjRSdEk3TmJwczBJQWo1aTRDZkpUQUMxYy81V0ZqQlJnakxBQUY4Q3dXOTIrM0FNaTVYdmlPbEx4ZVNlMS9pc0xtWXgvRloyQVgyZFk2Y3FhVkZ1b3RZREtoaGJmeUtrMUI0ZnpuNVRsZ1EzOXlWQ0xLMHlsblh6MFczZng4WG40NFBWdU1Tc1FzbU8wWTNLb2Uwdk1YWTlqczEzR09ZTlhyLzg9
```

- Upload the `shasum sig` file to the URL:
```shell
curl -T terraform-provider-oci_5.30.SHA256SUMS.sig https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6cWxwRGpxVC8xTW41MllHNFQxb1NNTlN4ODNGVklPVFAxc3U3b3RBb3hoR2pPa3pIVXhFaG50dlBtTlVsZjZNMm5kbzVJYVpGcnVLNS9YSllGQXhPYTVnOFFKczRXYXh6NGxIQys3d1NDekNyZ3dZcUdoTE1QdS9MaVY5cHRBRkZpTWNTL1YzKzV2UlZURDJJUk5aVnJVNzVDcFE3b2N6Mk9MZlBpSGxpQXRwTlJJWWY3M3p5NDJLZGFOS2tnN29YSHBEd09aa3dONExFSGRUWFlUZGV5aHZ3Um5sVDZJVjNLa1BSMG5uUGI1aldVcERJdGQzUm1IT2UxQitZRmpIK21oeklMd0lDOG1pZVdJYUp3WllPRzVWamRyZmhnMUNxejhGUUF3OTE0S1o0dWVXTg
```

- Create the payload file named `provider-version-payload-platform_darwin.json`:
```json
{
    "data": {
      "type": "registry-provider-version-platforms",
      "attributes": {
        "os": "darwin",
        "arch": "amd64",
        "shasum": "6571b434d23467f0edb47724df445a0cf39bb5c39f3df4e77efad3b0eacd390e",
        "filename": "terraform-provider-oci_5.30.0_darwin_amd64.zip"
      }
    }
  }
```

- Create the provider version platform:
```shell
curl -sS \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @provider-version-payload-platform_darwin.json \
  https://tfe67.aws.munnep.com/api/v2/organizations/test/registry-providers/private/test/oci/versions/5.30.0/platforms | jq '.'
```

- Save the value of the `provider-binary-upload` from the response:
```json
      "provider-binary-upload": "https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6OTNwVU5aSloyamZMRDQyR2xrWk9qZUxCMm1NRFgrVmdUR2lhS1RIZGI1QzlyYXlvbGU1NlRGV0VPeG15SkxGY2F3akw5WnhMdXlZdlVYeXRzdUd5Qjhrc0xPNUdnaCs5MjhTenhUTDY4ZVRRRnkwaGV3ZEFvRlFHMkpIUS9ieTM5d3hvMXc5TVA5UmM1eDF4Q0FGYnh0RWkyZTNwK05sUUc3OGlibXJxQUxxRTJZSXpwWE9TdVY3bGJQRFJ4RnBPMjltSkpKbmV0SC9STTlPNjNMYTRmZVc1elFpWTJjNGhBdU9YRXZZbC8waEdJNUxnZlh1SnMzLzBmZGRsQWc9PQ"
    }
```

- Upload the archived binary to the `provider-binary-upload` URL:
```shell
curl -T terraform-provider-oci_5.29.0_darwin_amd64.zip https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6OTNwVU5aSloyamZMRDQyR2xrWk9qZUxCMm1NRFgrVmdUR2lhS1RIZGI1QzlyYXlvbGU1NlRGV0VPeG15SkxGY2F3akw5WnhMdXlZdlVYeXRzdUd5Qjhrc0xPNUdnaCs5MjhTenhUTDY4ZVRRRnkwaGV3ZEFvRlFHMkpIUS9ieTM5d3hvMXc5TVA5UmM1eDF4Q0FGYnh0RWkyZTNwK05sUUc3OGlibXJxQUxxRTJZSXpwWE9TdVY3bGJQRFJ4RnBPMjltSkpKbmV0SC9STTlPNjNMYTRmZVc1elFpWTJjNGhBdU9YRXZZbC8waEdJNUxnZlh1SnMzLzBmZGRsQWc9PQ
```
- The provider for darwin is now uploaded. Do the same for linux
- Create the provider version platform for linux
```shell
curl -sS \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @provider-version-payload-platform_linux.json \
  https://tfe67.aws.munnep.com/api/v2/organizations/test/registry-providers/private/test/oci/versions/5.30.0/platforms | jq '.'
```

- Save the value of the `provider-binary-upload` from the response:
```json
      "provider-binary-upload": "https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6OTNwVU5aSloyamZMRDQyR2xrWk9qZUxCMm1NRFgrVmdUR2lhS1RIZGI1QzlyYXlvbGU1NlRGV0VPeG15SkxGY2F3akw5WnhMdXlZdlVYeXRzdUd5Qjhrc0xPNUdnaCs5MjhTenhUTDY4ZVRRRnkwaGV3ZEFvRlFHMkpIUS9ieTM5d3hvMXc5TVA5UmM1eDF4Q0FGYnh0RWkyZTNwK05sUUc3OGlibXJxQUxxRTJZSXpwWE9TdVY3bGJQRFJ4RnBPMjltSkpKbmV0SC9STTlPNjNMYTRmZVc1elFpWTJjNGhBdU9YRXZZbC8waEdJNUxnZlh1SnMzLzBmZGRsQWc9PQ"
    }
```

- Upload the archived binary to the `provider-binary-upload` URL:
```shell
curl -T terraform-provider-oci_5.30.0_linux_amd64.zip https://tfe67.aws.munnep.com/_archivist/v1/object/dmF1bHQ6djE6SVYvcUhJNG8rMlJTeEFYMy95NjRSR0FVdzNIbEtNdmNEVlFwaDM4bUVDRHoxdmdPUlRubXJQK3V6MG92QXNidFRqWTNLU2RZN3lYK0M0MEJuRmV0VjY1ajhPbDd2V2txN0VYSEg0d2dWY25oTjNTVVRNZklFTVhWWTRtUkxUV2lNUmxZcnJDeFdrNVd3VkJWUXhGRHg5dnV3OUYweGdkNDFuYmQrY0U0eVQ3MUhCRWtKc0k0anFJNFU0cnREdExyRzRDSWpGVmlqMFFsUjg3UmxBMXVCSmMrNElpZjNUQ1M2KzVVTkRRampCckI0azBvVXF2b2h5RE1aeXMwTlE9PQ
```

- The `oci` provider is now uploaded in the Terraform Cloud's private registry:

## Verify the private provider works 

- Create a new workspace with a CLI-driven workflow in your Terraform Cloud UI:

- Create a `main.tf` file locally with the following content:
```hcl
terraform {
  required_providers {
    oci = {
      source  = "tfe67.aws.munnep.com/test/oci"
      version = "5.30.0"
    }
  }

  cloud {
    hostname     = "tfe67.aws.munnep.com"
    organization = "test"

    workspaces {
      name = "oci_Test"
    }
  }
}




provider "oci" {
  # Configuration options 
}

```

- Login to Terraform Cloud through your terminal:
```shell
terraform login
```

- Initialize terraform to download the dependencies of the `oci` private provider:
```shell
terraform init

Initializing the backend...

Initializing provider plugins...
- Finding tfe67.aws.munnep.com/test/oci versions matching "5.30.0"...
- Installing tfe67.aws.munnep.com/test/oci v5.30.0...
- Installed tfe67.aws.munnep.com/test/oci v5.30.0 (self-signed, key ID CB3AD3052843121E)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

# Issues

In this case the shasum files were not uploaded correctly. I removed the version and did those steps again. Removed the version with the following API call

```shell
curl \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request DELETE \
  https://app.terraform.io/api/v2/organizations/patrickmunne/registry-providers/private/patrickmunne/myprovider/versions/0.1.0
```