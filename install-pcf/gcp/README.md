# PCF on GCP

![Concourse Pipeline](embed.png)

This pipeline uses Terraform to create all the infrastructure required to run a
3 AZ PCF deployment on GCP per the Customer[0] [reference
architecture](http://docs.pivotal.io/pivotalcf/1-10/refarch/gcp/gcp_ref_arch.html).

## Usage

This pipeline downloads artifacts from DockerHub (czero/cflinuxfs2 and custom
docker-image resources) and the configured Google Cloud Storage bucket
(terraform.tfstate file), and as such the Concourse instance must have access
to those. Note that Terraform outputs a .tfstate file that contains plaintext
secrets.

1. Within Google Cloud Platform, enable the following:
  * GCP Compute API [here](https://console.cloud.google.com/apis/api/compute_component)
  * GCP Storage API [here](https://console.cloud.google.com/apis/api/storage_component)
  * GCP SQL API [here](https://console.developers.google.com/apis/api/sqladmin.googleapis.com)
  * GCP DNS API [here](https://console.cloud.google.com/apis/api/dns)
  * GCP Cloud Resource Manager API [here](https://console.cloud.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)
  * GCP Storage Interopability [here](https://console.cloud.google.com/storage/settings)

2. Create a bucket in Google Cloud Storage to hold the Terraform state file, enabling versioning for this bucket via the `gsutil` CLI: `gcloud auth activate-service-account --key-file credentials.json && gsutil versioning set on gs://<your-bucket>`

3. Change all of the CHANGEME values in params.yml with real values. For the gcp_service_account_key, create a new service account key that has the following IAM roles:
  * Cloud SQL Admin
  * Compute Instance Admin (v1)
  * Compute Network Admin
  * Compute Security Admin
  * DNS Administrator
  * Storage Admin

4. [Set the pipeline](http://concourse.ci/single-page.html#fly-set-pipeline), using your updated params.yml:
  ```
  fly -t lite set-pipeline -p deploy-pcf -c pipeline.yml -l params.yml
  ```

5. Unpause the pipeline
6. Run `bootstrap-terraform-state` to bootstrap the Terraform .tfstate file. This only needs to be run once.
7. `upload-opsman-image` will automatically upload the latest matching version of Operations Manager
8. Trigger the `create-infrastructure` job. `create-infrastructure` will output at the end the DNS settings that you must configure before continuing.
9. Once DNS is set up you can run `configure-director`. From there the pipeline should automatically run through to the end.

### Tearing down the environment

There is a job, `wipe-env`, which you can run to destroy the infrastructure
that was created by `create-infrastructure`.

_**Note: This job currently is not all-encompassing. If you have deployed ERT you will want to delete ERT from within Ops Manager before proceeding with `wipe-env`, as well as deleting the BOSH director VM from within GCP. This will be done automatically in the future.**_

If you want to bring the environment up again, run `create-infrastructure`.

## Known Issues

### `create-infrastructure` job trying to delete SSL cert in use

When the `create-infrastructure` job runs, it may generate an error like this:
`google_compute_ssl_certificate.ssl-cert (destroy): 1 error(s) occurred:
google_compute_ssl_certificate.ssl-cert: Error deleting ssl certificate: googleapi: Error 400: The ssl_certificate resource 'projects/<redacted>/global/sslCertificates/<redacted>-gcp-lb-cert' is already being used by 'projects/<redacted>/global/targetHttpsProxies/<redacted>-gcp-https-proxy', resourceInUseByAnotherResource`
When this happens, after you've initially run create-infrastructure, update your params to supply the generated certs so they aren't recreated. Alternatively you can delete the ssl certificate using the following command:

```
gcloud compute ssl-certificates delete Name-of-Certificate
```

### `wipe-env` job
* The job does not account for installed tiles, which means VMs created by tile
  installations will be left behind and/or prevent wipe-env from completing.
  Delete the tiles manually prior to running `wipe-env` as a workaround.
* The job does not account for the BOSH director VM, which will prevent the job
  from completing. Delete the director VM manually in the GCP console as a
  workaround.

### Missing Jumpbox
* There is presently no jumpbox installed as part of the Terraform scripts. If
  you need to SSH onto the Ops Manager VM you'll need to add an SSH key from
  within GCP to the instance, and also add the `allow-ssh` tag to the network
  access tags.

### Cloud SQL Authorized Networks

There is a set of authorized networks added for the Cloud SQL instance which
has been modified to include 0.0.0.0/0. This is due to Cloud SQL only
managing access through public networks. We don't have a good way to keep
updated with Google Compute Engine CIDRs, and the Google Cloud Proxy is not
yet available on BOSH-deployed VMs. Thus, to allow Elastic Runtime access to
Cloud SQL, we allow 0.0.0.0/0. When a better solution comes around we'll be
able to remove it and allow only the other authorized networks that are
configured.

There is a (private, sorry) [Pivotal Tracker
story](https://www.pivotaltracker.com/n/projects/975916/stories/133642819) to
address this issue.
