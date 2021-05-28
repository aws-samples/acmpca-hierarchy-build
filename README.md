# AWS Certificate Manager (ACM) Private Certificate Authority (PCA) Hierarchy Deployment

This is a step-by-step guide to deploy an AWS Certificate Manager (ACM) Private Certificate Authority (PCA) hierarchy, which is a pre-requisite for you to automate the issuing and renewing of private certificates for [AWS services integrated with ACM](https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html):

- Elastic Load Balancing
- Amazon CloudFront
- AWS Elastic Beanstalk
- Amazon API Gateway
- AWS Nitro Enclaves (for EC2 instances connected to a Nitro Enclave)

You can use AWS CloudFormation to handle ACM Private Certificate provisioning (not part of this repository).

At the top of the CA (Certificate Authority) hierarchy is the Root CA, which is used to certify first-level Subordinate CAs. To create a Root CA, please use the CloudFormation template `acm-pca-root-ca.yaml`.

A Subordinate CA is certified by a parent CA, which is either the Root CA or another Subordinate CA. To create a Subordinate CA, please use the CloudFormation template `acm-pca-sub-ca.yaml`.

## Prerequisites

- Private Hosted Zone for the Certificate Revocation List (CRL) Cname
  - Each ACM PCA stores its CRL in an S3 bucket under object prefix `crl/<pca-id>.crl`. It is a best practice to use a Canonical Name (Cname) instead of exposing the S3 bucket, e.g., `http://<crl-cname>/crl/<pca-id>.crl`. The CRL Cname record is to be created in the CRL Private Hosted Zone, e.g., `crl.example.int` created in private hosted zone `example.int`.
  - Note: The CRL private hosted zone does not need to be related to the domain names the private certificates are issued in. e.g., A certificate of Common Name (CN) `myapp.example.int` may have a CRL Cname being `crl.pca.corporate.com`. i.e., You may share CRL Cname across CAs.
  - You may use CloudFormation template `private-hosted-zone.yaml` to create a private hosted zone if you do not already have one.
- Execution role (Optional)
  - You may use CloudFormation template `execution-role.yaml` to create the execution role with permissions required to create the ACM PCA stack.

## Step-by-step guide

All CloudFormation templates are under `cfn-templates/`.

### Root CA

1. Create the Root CA stack
   - CloudFormation template: `acm-pca-root-ca.yaml`
   - CaChainS3BucketName: S3 Bucket to be created for CloudFormation to hold the CA Chain and Template snippet
   - CaValidityValue and CaValidityType: How long the CA is valid for; Recommend to use a long period for the Root CA; default is 10 YEARS
   - CrlCname: CRL Canonical Name (Cname) to the CRL hosting S3 bucket (a globally unique name), e.g., crl.pca.example.int
   - CrlPrivateZoneId: Zone ID of the Private Hosted Zone where CrlCname is hosted
2. Wait until the stack creation is complete
3. From the Resources tab, note down  `<root-ca-arn>` from the "Physical ID" column of the "RootCa" row, and use it to certify first level Subordinate CAs
4. Verify the Root CA: `bin/acm-pca-listPcas Type Status CertificateAuthorityConfiguration.Subject.CommonName`

### Subordinate CA

1. Create the Subordinate CA stack
   - CloudFormation template: `acm-pca-sub-ca.yaml`
   - CaChainS3BucketName: Existing S3 Bucket for CloudFormation to hold the CA Chain and Template snippet
   - CaValidityValue and CaValidityType: How long the CA is valid for; A subordinate CA must not outlive its parent CA; i.e., must expire before its parent CA does
   - CertifyingCaArn: `<parent-ca-arn>`
   - CrlCname: CRL Canonical Name (Cname) to the CRL hosting S3 bucket (a globally unique name), e.g., crl.pca.example.int
   - CrlCnameIsNew: Select YES if you want to create a new CRL Cname, or NO if the specified CRL Cname already exists (e.g., a shared CrlCname)
   - CrlPrivateZoneId: Zone ID of the Private Zone where CrlCname is hosted
   - SubCaPathLen: How many levels of Subordinate CAs can be certified under this new Subordinate CA; default is 0 (no Subordinate CAs under this)
2. Wait until the stack creation is complete
3. From the Resources tab, note down  `<sub-ca-arn>` from the "Physical ID" column of the "SubCa" row, and use it to certify next level of Subordinate CAs if required
4. Verify the Subordinate CA: `bin/acm-pca-listPcas Type Status CertificateAuthorityConfiguration.Subject.CommonName`
5. Repeat the above to create more subordinate CAs if required

## Useful Tips and Tools

Under the `bin/` directory, there are some useful Shell scripts as a simplified alternative to using AWS CLI. They can also serve as a guide to learn ACM related AWS CLI and OpenSSL commands.

Generally, entering the command without parameters will show the help text. For commands that allow no parameters, option `-?` will show the help text (e.g., `bin/acm-pca-listPcas -?`).

### List all private CAs

`bin/acm-pca-listPcas Type Status CertificateAuthorityConfiguration.Subject.CommonName`

Use option `-j` to show all details in JSON, and `-a` to include deleted CAs as well.

### Verify the private CAs from the AWS Console
1. Open the Certificate Manager console
2. Click the "Private CAs" menu
3. Find the "Active" Private CAs
4. Check if the PCA ARN matches the Resources created in the CloudFormation stack

### Verify the CA Certificate

Here `<ca-name>` is an arbitrary name that you want to use to name the output file.

```
bin/acm-pca-getCaCertificate <pca-arn> > <ca-name>-ca.cert
```

Use `bin/certText <ca-name>-ca.cert | less` to show the certificate details.

### Verify the CA Chain

Here `<ca-name>` is an arbitrary name you want to use for the CA file path.

```
bin/acm-pca-getCaCertificate -c <pca-arn> > <ca-name>-chain.cert
```

You may also download the CA Certificate and Chain in one file from the S3 bucket.

```
aws s3 cp s3://<ca-chain-s3-bucket>/pca/<pca-id>-chain.pem <ca-name>-chain.cert
```

Use `bin/chainText <ca-name>-chain.cert | less` to show the chain. Use option `-a` to show details of each certificate in the CA chain.

Note: The CA Chain of the first-level subordinate CAs is the same as the Root CA Certificate. e.g., `root-ca.cert` is the same as `sub-level-1-chain.cert`.

### Verify issuing capability

#### Request a certificate (with default settings)

Request a certificate using the default settings: a 13-month validity and a simple Subject `CN = <domain>`.

```
bin/acm-requestCertificate <pca-arn> <domain>
```
Output
```
{
    "CertificateArn": "<new-cert-arn>"
}
```

Get the certificate

```
bin/acm-getCertificate <new-cert-arn> > <domain>.cert
bin/certText <domain>.cert | less
```
Note down the serial number, e.g., ff:d8:fc:70:5e:2f:3e:1c:fc:58:4e:fa:73:32:c6:a7

Find the certificate on the Certificate manager console and verify the ARN and serial number.

#### Issue a certificate

Issue a certificate by generating the private key and certificate signing request (CSR) locally.

```
bin/acm-pca-issueCertificate -s <subject> -v <validity> <pca-arn> <domain>
```

Output

```
{
    "CertificateArn": "<new-cert-arn>"
}
```

Get the certificate

```
bin/acm-pca-getCertificate <new-cert-arn> > <domain>.cert
bin/certText <domain>.cert | less
```

Note down the serial number, e.g., 79:57:43:a0:4a:9a:2c:33:b4:8f:96:8f:bb:00:f9:93

Certificates issued using `bin/acm-pca-issueCertificate` are *not* managed by ACM, which means such certificates

- are not renewable by ACM (and does not appear on the ACM console)
- with no private key kept in ACM (If you tried to export the certificate, it will show an error that the certificate ARN is not valid.)
- with a certificate ARN in the format of `<ca-arn>/certificate/<cert-id>` (prefixed by Prinvate CA ARN)
- can be retrieved using `bin/acm-pca-getCertificate` (instead of `bin/acm-getCertificate` because the Certificate ARN does not exist in ACM)

Here is the full help text of `acm-pca-issueCertificate`:

```
Usage: acm-pca-issueCertificate [-s subject] [-v validity] [-a signAlgo] [-k keyLen] [-m messageDigest] pca_arn domain
  Issue a certificate signed by a Private CA, i.e., Certificate ARN starting with arn:aws:acm-pca:
  Note 1: Certificates issued by a Private CA are not listed on ACM Console.
  Note 2: Two files will be generated: <domain>.key and <domain>.csr
        -?          Show this help text
        pca_arn     ARN of Private CA used to issue the certificate
        domain      Common Name (CN) in the certificate Subject
        -v validity Validity of the certificate; e.g., 13m (default), 10y, 365d etc
        -s subject  Full Subject of the certificate; default /C=AU/ST=NSW/L=Sydney/O=My Organisation/OU=My Organisational Unit/CN=<domain>
        -a signAlgo Signing algorithm; default SHA512WITHRSA
  Settings for openssl
        -k keyLen   Key Length; default 4096
        -m md       Message digest to sign the request with: md1, sha1, sha256; default sha256
```

#### Verify CRL

```
curl http://<crl-cname>/crl/<pca-id>.crl --output <ca-name>.crl
```

If you cannot resolve `<crl-cname>`, you may check the Cname record value on Route53 and substitute `<crl-cname>` with the Cname value in your curl command.

You may also download the CRL from the S3 bucket.

```
aws s3 cp s3://<crl-cname>/crl/<pca-id>.crl <ca-name>.crl
```

View the CRL file
```
bin/crlText <ca-name>.crl | less
```
The following line indicates that there are no certificates being revoked.
```
No Revoked Certificates.
```

### Revoke a certificate
```
bin/acm-pca-revokeCertificate <pca-arn> <reason> <cert-serial>
```
Reason code is one of the following:
- AFFILIATION_CHANGED
- CESSATION_OF_OPERATION
- A_A_COMPROMISE
- PRIVILEGE_WITHDRAWN
- SUPERSEDED
- UNSPECIFIED
- KEY_COMPROMISE
- CERTIFICATE_AUTHORITY_COMPROMISE

Please allow up to 30 minutes for the CA to update the CRL. After the CRL is updated, you may download the CRL file to verify.

```
aws s3 cp s3://<crl-bucket>/crl/<pca-id>.crl <ca-name>.crl
```

View the CRL file.
```
bin/crlText <ca-name>.crl | less
```
Under "Revoked Certificates" the revoked certificate will be shown. For example,
```
Revoked Certificates:
    Serial Number: 795743A04A9A2C33B48F968FBB00F993
        Revocation Date: Mar  3 01:43:35 2021 GMT
        CRL entry extensions:
            X509v3 CRL Reason Code:
                Superseded
```

Note: Revocations will not be shown on the ACM console. You can download and verify the CRL using the above tools.

### Delete a CA

*Warning*: ACM Private CA costs US$400/month pro-rated. It is recommended to delete test Private CAs when it is no longer needed.

- Delete the private CA stack first
  - The Private CA will be in the DISABLED state.
- Delete the private CA using `bin/acm-pca-deletePca <pca-arn>`
  ```bash
  Private CA: ...
  Old Status: DISABLED
  New Status: DELETED
  Action: To be deleted in 7 days
  ```

Please note that deleting a private CA does not invalidate the certificates the private CA has already signed before the deletion.
If you want to revoke some certificates signed by the deleted CA, you need to do so before you delete the private CA, and keep the CRL S3 Bucket website hosting intact, so that the CRL URL remains accessible for revocation verification.

A quick way to revoke all certificates signed by any private CAs in the hierarchy is to have the root CA revoke the second-level private CAs and keep the root CA CRL S3 Bucket intact. After that, you may remove all private CAs in the hierarcy.

For the above reason, in the CloudFormation templates, the deletion policy of the CRL Cname and S3 Bucket is set to `Retain`, so that private CA stack deletion will not affect the CRL. If you no longer need the CRL of a deleted private CA (e.g., after all certificates the deleted private CA revoked have expired), you may delete the CRL Cname and S3 Bucket manually.
