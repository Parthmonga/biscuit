# Biscuit

[![Build Status](https://travis-ci.org/dcoker/biscuit.svg)](https://travis-ci.org/dcoker/secrets)

Biscuit is a simple key-value store for your infrastructure secrets.

## Is Biscuit right for me?

Biscuit is most useful to teams already using AWS and IAM to manage their 
infrastructure. If that describes your team, then Biscuit might be useful to 
you if you answer "yes" to any of these questions:

* Do you live in constant fear of accidentally committing infrastructure secrets to source control?
* Do you commit private keys to your source repository?
* Do you share passwords with other developers?
* Do you want to manage secrets securely across multiple regions?

### Features

* Provides a simple key/value CLI to secure storage.
* Secrets can live alongside with your code in source control.
* Operates with KMS keys across multiple regions.
* Facilitates management of AWS IAM Policies, KMS Policies, and KMS Grants across multiple regions.
* Local encryption using AES-GCM-256 or Secretbox (NaCL).
* Offline mode: Using the "testing" key manager, you can use Biscuit in
  test environments without changing your code and without network 
  dependencies.

### Feature Comparison

| Package                                             | Requires a server? | Multi-region | HA  | Rotation | Storage  | AWS KMS  | Principals | Web UI |
|:----------------------------------------------------|:-------------------|:-------------|:----|:---------|:---------|:---------|:-----------|:-------|
| Biscuit                                             | No                 | Yes          | Yes | No       | File     | Required | AWS Only   | No     |
| [Credstash](https://github.com/fugue/credstash)     | No                 | No           | Yes | No       | DynamoDB | Required | AWS Only   | No     |
| [Lyft Confidant](https://github.com/lyft/confidant) | Yes                | No           | No  | No       | DynamoDB | Required | AWS Only   | Yes    |
| [Hashicorp Vault](https://www.vaultproject.io)      | Yes                | Yes          | Yes | Yes      | Varied   | Optional | Multiple   | No     |

## Quick Start

### Installing

#### From Binaries

coming soon

#### From Source

If you have Golang 1.6+ installed, you can install with:

```
go get -v github.com/dcoker/biscuit
```

### Setup

```shell
# Verify that your AWS credentials are readable.
biscuit kms get-caller-identity

# Provision a KMS Key w/useful defaults in us-east-1, us-west-1, 
# and us-west-2 and create a secrets.yml file.
biscuit kms init -f secrets.yml

# Store the launch codes.
biscuit put -f secrets.yml launch_codes 0000

# Decrypt the launch codes.
biscuit get -f secrets.yml launch_codes
```

Next steps: examine `secrets.yml` in your favorite text editor, and run 
`biscuit --help` to learn about additional commands.

### Uninstalling

Done already?

The `biscuit kms init` step above may have created a KMS Key and some
associated policies using CloudFormation. You can remove those by
running:

```shell
biscuit kms deprovision
rm secrets.yml
```

Note: any biscuit files you created before deprovisioning will no longer
be readable.

### Glossary

The **secret** is the plaintext value which you wish to protect.

A **label** is a short alphanumeric string that identifies a set of keys
across multiple AWS regions. It is present in CloudFormation stack names
(`biscuit-label`) and in KMS key aliases (`alias/biscuit-label`).

The **key manager** is responsible for the provisioning of encryption
keys. The encryption keys generated by the key manager are used to
encrypt the **secret**. An encrypted version of the encryption key -- 
decryptable only by KMS -- is stored as the **key ciphertext**.

Secret **values** consist of the information necessary for the **key
manager** to provide the plaintext encryption key to decrypt a
**ciphertext** for a named secret. Values consist of a Key ID (a
string, meaningful to the key manager), an indicator of which key manager is
in use (string), an algorithm (string), the key ciphertext (the
encrypted key, base64), and the ciphertext (base 64). Here is an example
of a value named `api_key`:

```yaml
api_key:
- key_id: arn:aws:kms:us-west-1:123456789012:key/37793df5-ad32-4d06-b19f-bfb95cee4a35
  key_manager: kms
  algorithm: secretbox
  key_ciphertext: CiA3edlKfUWXVgiDDuzbz95S/pkM8grwRsYkjRoURv0LGhKnAQEBAQB4N3nZSn1Fl1YIgw7s28/eUv6ZDPIK8EbGJI0aFEb9CxoAAAB+MHwGCSqGSIb3DQEHBqBvMG0CAQAwaAYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAw4OEtFZrisfC3xJHACARCAO+HJpH4bWD/MF9BYjBvl5ztcezTNxo5SPeAOKJ3Z8Pff2vh1uCZhEEjxnF7t1tqTma8oeESuu2vpPiZp
  ciphertext: YsI/4Qnzpu+Vm+JP4LhnO8Y3dSoz61/vKHBXGVI1pVAUCjMhvjb9ohcdjA==
```

**Names** identify a value. When using AWS KMS, names are
encoded into the **encryption context** and must be provided by the
process that decrypts the value.

A key **template** is a special entry in the .yml file which tells the
biscuit tool how to handle values that are added to that file. This is
an example template:

```yaml
_keys:
- key_id: arn:aws:kms:us-west-1:123456789012:key/37793df5-ad32-4d06-b19f-bfb95cee4a35
  key_manager: kms
  algorithm: secretbox
- key_id: arn:aws:kms:us-west-2:123456789012:key/c0045b15-9880-4b17-84da-a35760e8a16f
  key_manager: kms
  algorithm: secretbox
- key_id: arn:aws:kms:us-east-1:123456789012:key/d1c5a8e3-adfb-4f79-af0b-cde9f1a31292
  key_manager: kms
  algorithm: secretbox
```

## IAQ

### How much does this cost?

Biscuit requires one key in each region you wish to use. See 
[AWS Key Management Service Pricing](https://aws.amazon.com/kms/pricing/) for pricing.

### Can I use it in a single AWS region?

Yes. Use the `-r` or `BISCUIT_REGIONS` environment variable to set the region.

### What do I do with the .yml file after it is created?

The .yml file is safe for committing to version control, your CI system,
embedding in native binaries, copying to S3, publishing in a newspaper,
etc. You can deploy these to your production servers by whatever
mechanism is most appropriate for your environment, but by far the
easiest way to start is to simply include it in your deployments in the
same way you would a configuration file.

### Once I've created a value, how do I let AWS resources decrypt it?

You can use KMS Grants, KMS Key Policies, or IAM Policies to manage access 
to the secrets.

#### KMS Grants

KMS Grants enable you to delegate access to specific KMS operations to some
AWS principal. Often your AWS resources will be running with an IAM Role, and 
thus often the easiest thing to do is to use KMS Grants to allow your IAM Roles
to decrypt the appropriate values.

Biscuit will create and retire those grants for you.

Here's an example:

```shell
biscuit kms grants create --grantee-principal role/webserver -f secrets.yml launch_codes
biscuit kms grants create --grantee-principal user/gordon -f secrets.yml launch_codes
biscuit kms grants list -f secrets.yml launch_codes
```

Once created, the principals listed (and any EC2 instances or Lambda 
functions running under those roles) can decrypt the `launch_codes`
secret.

You can also retire grants when they are no longer useful:

```shell
biscuit kms grants list -f secrets.yml launch_codes
biscuit kms grants retire -f secrets.yml --grant-name biscuit-ff8102edc8 launch_codes
```

#### KMS Key Policies

KMS Keys have their own policies. See 
[Key Policies](http://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html) 
for more details. If you have just a few users, this is possibly the easiest
mechanism to use to control access. You can run `biscuit kms edit-key-policy` to 
edit the policy document across all of your regions at once.

#### IAM Policies

IAM Policies are attached to a myriad of AWS entities, and they can also be 
used to enable access to KMS operations. See 
[Key Policies](http://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html) for more details.

Example: You have a server running with a static AWS access key and secret key. You can give that
server the ability to decrypt all values encrypted under a set of keys by attaching a standard user policy, 
specifying `kms:Decrypt` as the Action and the full key ARNs in the Resource field.

Note: IAM Policies are global entities, whereas KMS Keys are unique per region. Thus if you have a 3-region 
configuration, any IAM Policies that explicitly grant access to KMS Keys will need to list all 3 
region-specific key ARNs.

If you wish to disallow IAM Policies from controlling access to your keys,
you can do so by passing `--disable-iam-policies` to `kms init`. When IAM
Policies are disabled, the only way to control access to keys is via Grants and KMS Key 
Policies. For more information on how this works, see the CloudFormation 
template in the source repository and the Key Policies doc linked above.

### How do I keep my development and production keys separate?
 
Biscuit tracks keys across regions by using a label. Labels are embedded 
into the name of the CloudFormation stack and a KMS Key Alias in each region. 
The default behavior is to use the `default` label, but you can change this 
by passing the `-l` flag. Example: `biscuit kms init -l development`
 
Labels are not persisted with the values and are not visible in the .yml 
files. They are an organizational tool to facilitate managing keys across 
regions, and are passed as parameters to various commands.

Here are some common scenarios:

```shell
# Create key in a single-region, and allow developers to do whatever they want. 
biscuit kms init -r us-west-2 -l development -f development.yml --administrators role/developers --users role/developers
biscuit put -f development.yml ssl_key -i selfsigned.key

# Create keys in three regions, and allow a limited set of people to administer it,
# and allow developers (and gordon) to read and write the secrets.
biscuit kms init -l production -f production.yml --administrators role/prod-keymaster --users role/developers,user/gordon
biscuit put -f production.yml ssl_key -i wildcard.key

# Create a file that doesn't use encryption at all.
biscuit put -f unittest.yml -a none database_password testing
```

### What's the difference between an "administrator" and a "user"?

Biscuit installs a KMS Key Policy similar to the default policy 
recommended by AWS. This creates a distinction between 
["administrators"](http://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html#key-policy-default-allow-administrators) 
and ["users"](http://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html#key-policy-default-allow-users).

Administrators can administer the key but not necessarily encrypt or decrypt 
with that key. They could, if they wished, replace the key policy with one 
that does allow them encrypt and decrypt, but that would be unusual.

Users can encrypt, decrypt, and generate ephemeral data keys. In most 
cases, you'll want to make your development team a "user" and 
possibly also "administrators". 

If you use `biscuit kms init` to create your keys, you can use the 
`--administrators` and `--users` flag to set membership. If you have already
created the keys but want to change the policy, use the interactive
`biscuit kms edit-key-policy` to apply changes to all regions simultaneously.

### Does Biscuit support multi-factor authentication?

Yes. IAM Policies and the KMS Key Policies support the `aws:MultiFactorAuthPresent` condition.

Biscuit does not enforce any access control on its own. All enforcement is
implemented by AWS. Any features that are available in the IAM Policies, KMS Key 
Policy documents, or KMS Grants are available to you. 

Biscuit does not expose command line flags to use all of the features, but you 
can edit the various policies after they are created, or override the default 
CloudFormation template.

Biscuit is known to work well with [awsmfa](https://pypi.python.org/pypi/awsmfa).

### I manually edited the .yaml file and changed the name of a value and now it won't decrypt. What's wrong?

The `kms` key manager annotates the ciphertext with an
[EncryptionContext](http://docs.aws.amazon.com/kms/latest/developerguide/encryption-context.html)
containing the name of the value. If you change the name of a value,
then the decrypting process will provide the wrong name to the decrypt
operation and the decrypt will fail. If you wish to change the name of a
secret, re-encrypt it using the new name instead.

### I want to change something about the CloudFormation template. What do I do?

The `biscuit kms init` command allows you to override the built-in
CloudFormation template with your own via the
`--cloudformation-template-url` parameter. 

### My account administrator does not let me create CloudFormation stacks, help!

The only expectations that Biscuit has about your KMS configuration is that the 
keys have aliases of the form `alias/biscuit-{label}` and that you have sufficient 
permissions to operate on the KMS keys. You can create the keys using whatever
process is compatible with your organization's policies.

### How do I rotate the values?

Biscuit considers the rotation of secrets (such as database passwords)
to be application-specific features and thus does not have any native
support for it. However, you can
implement a rotation scheme appropriate for your situation simply by
serializing that state as the secret and then read it with your 
application-specific rotation behaviors. 

Here is an example using JSON and [jq](https://stedolan.github.io/jq/):

```shell
biscuit put -f secrets.yml database_passwords '{"1": "pass1", "2": "pass2"}'
biscuit get -f secrets.yml database_passwords | jq -r '.[to_entries | map(.key) | map(tonumber) | max | tostring]'
```

