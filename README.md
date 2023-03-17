# Temporal Money Transfer

This is a simple project for demonstrating Temporal with the Go SDK and [Temporal Cloud](https://cloud.temporal.io).

This project is based on [money-transfer-project-template-go](https://github.com/temporalio/money-transfer-project-template-go) and the corresponding tutorial here: https://learn.temporal.io/getting_started/go/first_program_in_go/

## Basic instructions

### Step 0: Temporal Cloud

This guide assumes you already have access to a Temporal Cloud account, as described in [How to get started with Temporal Cloud](https://docs.temporal.io/cloud/how-to-get-started-with-temporal-cloud)

### Step 1: Clone this Repository

In a terminal instance, clone this repo.

```bash
git clone https://github.com/pvsone/temporal-money-transfer
cd pvsone-money-transfer
```

### Step 2: Issue CA Certificates

The Temporal docs provide options for [Issuing CA Certificates](https://docs.temporal.io/cloud/how-to-get-started-with-temporal-cloud#issue-ca-certificates)

For this guide, we will use [certstrap](https://github.com/square/certstrap) to create a root CA and end-entity certificate.

_Note: In Step 3 of this guide, we will **Create a Namespace**, during which we will supply a value for 
the Namespace Name.  We will use the Namespace Name as the Common Name of the end-entity certificate
that is created here in Step 2._

```bash
# set the Namespace Name for use as the Common Name in the certstrap commands below
export NAMESPACE_NAME=my-namespace

# Initialize a new certificate authority
certstrap init --common-name "My Cert Auth"

# Request a certificate, with a common name equal to the namespace name
certstrap request-cert --common-name ${NAMESPACE_NAME}

# Sign certificate request and generate the end-entity certificate
certstrap sign ${NAMESPACE_NAME} --CA "My Cert Auth"
```

### Step 3: Create a Namespace

Follow the steps at [Create a Namespace in Temporal Cloud](https://docs.temporal.io/cloud/how-to-manage-namespaces-in-temporal-cloud/#create-a-namespace-in-temporal-cloud).  In addition set the following field values:
* Set **CA Certificates** field value to the contents of the file `out/My_Cert_Auth.crt`.
* Set **Certificate Filters** > **Filter 1** > **Common Name** field value to the value of $NAMESPACE_NAME defined above, e.g. `my-namespace`

The **Certificate Filter** ensures that only the end-entity certificate that we created in Step 2
can be used to access this Namespace.  Without the filter in place then _any_ certificate signed 
and generated by the "My Cert Auth" could be used to access this Namespace.

After successful creation, the Namespace will be given a unique Id of the form `<namespace_name>.<account_id>`, e.g. `my-namespace.a1bc2`.  Export this value as environment variable in your shell for use in commands in subsequent steps.

```bash
# replace 'a1bc2' with your account id
export ACCOUNT_ID=a1bc2
export NAMESPACE_ID=${NAMESPACE_NAME}.${ACCOUNT_ID}
```

### Step 4: Run the Workflow

```bash
go run start/main.go \
  -address ${NAMESPACE_ID}.tmprl.cloud:7233 \
  -namespace ${NAMESPACE_ID} \
  -tls_cert_path ./out/${NAMESPACE_NAME}.crt \
  -tls_key_path ./out/${NAMESPACE_NAME}.key
```

Observe that Temporal Cloud Web UI reflects the workflow, but it is still in "Running" status. This is because there is no Workflow or Activity Worker yet listening to the `TRANSFER_MONEY_TASK_QUEUE` task queue to process this work.

### Step 5: Run the Worker

In another terminal instance, run the worker. Notice that this worker hosts both Workflow and Activity functions.

```bash
# export environment variables 
export NAMESPACE_NAME=my-namespace
export ACCOUNT_ID=a1bc2
export NAMESPACE_ID=${NAMESPACE_NAME}.${ACCOUNT_ID}

go run worker/main.go \
  -address ${NAMESPACE_ID}.tmprl.cloud:7233 \
  -namespace ${NAMESPACE_ID} \
  -tls_cert_path ./out/${NAMESPACE_NAME}.crt \
  -tls_key_path ./out/${NAMESPACE_NAME}.key
```

Now you can see the workflow run to completion. 


## Let's Encrypt Certificates
_Warning: You probably don't want to use Let's Encrypt to create client certificates.  While it will work technically, it is advisable to create your own Certificate Authority to use for issuing and managing client certificates.  See https://community.letsencrypt.org/t/can-i-create-client-certificates-for-a-received-letsencrypt-certificate/78627_

To use [Let's Encrypt](https://letsencrypt.org/), I repeated the steps with the following modifications:

### Step 2: Issue CA Certificates

Use [acme.sh](https://github.com/acmesh-official/acme.sh) to generate the certs from a shell.  For me, I have a personal domain (pvslab.net) registered with AWS Route53, so I issued the cert using the [ACME Route53 DNS API](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#10-use-amazon-route53-domain-api) based verification.

```bash
export AWS_ACCESS_KEY_ID=XXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXX

acme.sh --issue \
  --dns dns_aws \
  --domain sullivan-dev.tcld.pvslab.net \
  --server letsencrypt \
  --preferred-chain "ISRG"
```

The `--server` and `--preferred-chain` values are important to generate certs fully compatible with Temporal Cloud. I imagine there are other values that could be used here, but these are the ones that worked for me.

The important files generated from the `acme.sh` command are:
* Cert: Written to ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.cer
* Key: Written to ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.key
* Intermediate CA cert: Written to ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/ca.cer

The Intermeidate CA cert must be combined with the Root CA cert before uploading to Temporal Cloud.  Get the CA for the ISRG Root from the [certificates](https://letsencrypt.org/certificates/) page on Let's Encrypt or using the following command:

```bash
# get the ISRG self-signed Root CA from letsencrypt
curl -O https://letsencrypt.org/certs/isrgrootx1.pem
```

Then combine Intermediate CA with the Root CA in a single PEM file
```bash
# combine the Intermediate CA with the ISRG Root CA
cat ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/ca.cer isrgrootx1.pem > cacerts.pem
```

### Step 3: Update the Namespace

Update the Namespace configuration with the following field values:
* Set **CA Certificates** field value to the contents of the file `cacerts.pem`.
* Set **Certificate Filters** > **Filter 1** > **Common Name** field value to `sullivan-dev.tcld.pvslab.net`

### Step 4: Run the Workflow

```bash
go run start/main.go \
  -address ${NAMESPACE_ID}.tmprl.cloud:7233 \
  -namespace ${NAMESPACE_ID} \
  -tls_cert_path ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.cer \
  -tls_key_path ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.key
```

### Step 5: Run the Worker

```bash
go run worker/main.go \
  -address ${NAMESPACE_ID}.tmprl.cloud:7233 \
  -namespace ${NAMESPACE_ID} \
  -tls_cert_path ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.cer \
  -tls_key_path ~/.acme.sh/sullivan-dev.tcld.pvslab.net_ecc/sullivan-dev.tcld.pvslab.net.key
```
