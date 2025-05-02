gcloud
======

## `gcloud`

- free tier
  - <https://cloud.google.com/free/docs/free-cloud-features#free-tier>
  - compute engine
    - `e2-micro`
    - `us-west1`
    - 30 GB-months standard persistent disk
- <https://cloud.google.com/sdk/docs>
  - untar the tarball
  - `install.sh` or use `google-cloud-sdk/bin/gcloud` directly
- components
  - `gcloud components list`
  - `gcloud components update`
- init
  - `gcloud init`
- auth
  - `gcloud auth list` lists accounts
  - `gcloud auth login` logs in
- compute
  - os login
    - `gcloud compute project-info add-metadata --metadata=enable-oslogin=true`
    - `gcloud compute os-login ssh-keys add --key-file ~/.ssh/id_rsa.pub`
    - the user name is the email with special characters (`@` or `.`) replaced
      by `_`
  - quick ssh
    - `gcloud compute ssh <vm-instance>`
    - this adds the ssh key to the vm temporarily and ssh to it

