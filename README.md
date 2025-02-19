# Credentials Fetcher

`credentials-fetcher` is a Linux daemon that retrieves gMSA credentials from Active Directory over LDAP. It creates and refreshes kerberos tickets from gMSA credentials. Kerberos tickets can be used by containers to run apps/services that authenticate using Active Directory.

This daemon works in a similar way as ccg.exe and the gMSA plugin in Windows as described in - https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts#gmsa-architecture-and-improvements

### How to install and run

On [Fedora 36](_https://alt.fedoraproject.org/cloud/_) and similar distributions, the binary RPM can be installed as
`sudo dnf install credentials-fetcher`.
You can also use yum if dnf is not present.
The daemon can be started using `sudo systemctl start credentials-fetcher`.

On Enterprise Linux 9 ( RHEL | CentOS | AlmaLinux ), the binary can be installed from EPEL. To add EPEL, see the [EPEL Quickstart](_https://docs.fedoraproject.org/en-US/epel/#_quickstart_).
Once EPEL is enabled, install credentials-fetcher with
`sudo dnf install credentials-fetcher`.

For other linux distributions, the daemon binary needs to be built from source code.

## Development

### Prerequisites

- Active Directory server ( Windows Server )
- Linux instances or hosts that are domain-joined to Active Directory
  - [EC2 Linux containers on Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/linux-gmsa.html#linux-gmsa-considerations) provides the option of domainless gMSA and joining each instance to a single domain
  - [Linux containers on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-linux-gmsa.html#fargate-linux-gmsa-considerations) must use domainless gMSA
- gMSA account(s) in Active Directory - Follow instructions provided to create service accounts - https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts
- Required packages as mentioned in RPM spec file.
- Create username ec2-user or modify the systemd unit file.

#### Create credentialspec associated with gMSA account:

- Create a domain joined windows instance
- Install powershell module - "Install-Module CredentialSpec"
- New-CredentialSpec -AccountName WebApp01 // Replace 'WebApp01' with your own gMSA
- You will find the credentialspec in the directory
  'C:\\Program Data\\Docker\\Credentialspecs\\WebApp01_CredSpec.json'

#### Standalone mode

To start a local dev environment from scratch:

```
* Clone the Git repository.
* cd credentials-fetcher && mkdir build
* cd build && cmake ../ && make -j
* ./credentials-fetcher to start the program in non-daemon mode.
```

#### Testing

To communicate with the daemon over gRPC, install grpc-cli. For example
`sudo yum install grpc-cli`

##### AddKerberosLease API:

Note: APIs use unix domain socket

```
Invoke the AddkerberosLease API with the credentialsspec input as shown:
grpc_cli call {unix_domain_socket} AddKerberosLease "credspec_contents: '{credentialspec}'"

Sample:
grpc_cli call unix:/var/credentials-fetcher/socket/credentials_fetcher.sock
AddKerberosLease "credspec_contents: '{\"CmsPlugins\":[\"ActiveDirectory\"],\"DomainJoinConfig\":{\"Sid\":\"S-1-5-21-4217655605-3681839426-3493040985\",
\"MachineAccountName\":\"WebApp01\",\"Guid\":\"af602f85-d754-4eea-9fa8-fd76810485f1\",\"DnsTreeName\":\"contoso.com\",
\"DnsName\":\"contoso.com\",\"NetBiosName\":\"contoso\"},\"ActiveDirectoryConfig\":{\"GroupManagedServiceAccounts\":[{\"Name\":\"WebApp01\",\"Scope\":\"contoso.com\"}
,{\"Name\":\"WebApp01\",\"Scope\":\"contoso\"}]}}'"

* Response:
  lease_id - unique identifier associated to the request
  created_kerberos_file_paths - Paths associated to the Kerberos tickets created corresponding to the gMSA accounts
```

##### DeleteKerberosLease API:

```
Invoke the Delete kerberosLease API with lease id input as shown:
grpc_cli call {unix_domain_socket} DeleteKerberosLease "lease_id: '{lease_id}'"

Sample:
grpc_cli call unix:/var/credentials-fetcher/socket/credentials_fetcher.sock DeleteKerberosLease "lease_id: '${response_lease_id_from_add_kerberos_lease}'"

* Response:
    lease_id - unique identifier associated to the request
    deleted_kerberos_file_paths - Paths associated to the Kerberos tickets deleted corresponding to the gMSA accounts

```

### Logging

Logs about request/response to the daemon and any failures.

```
journalctl -u credentials-fetcher
```

#### Default environment variables

| Environment Key             | Examples values                    | Description                                                                                  |
| :-------------------------- | ---------------------------------- | :------------------------------------------------------------------------------------------- |
| `CF_KRB_DIR`                | '/var/credentials-fetcher/krbdir'  | _(Default)_ Dir path for storing the kerberos tickets                                        |
| `CF_UNIX_DOMAIN_SOCKET_DIR` | '/var/credentials-fetcher/socket'  | _(Default)_ Dir path for the domain socker for gRPC communication 'credentials_fetcher.sock' |
| `CF_LOGGING_DIR`            | '/var/credentials-fetcher/logging' | _(Default)_ Dir Path for log                                                                 |
| `CF_TEST_DOMAIN_NAME`       | 'contoso.com'                      | Test domain name                                                                             |
| `CF_TEST_GMSA_ACCOUNT`      | 'webapp01'                         | Test gMSA account name                                                                       |

#### Runtime environment variables

| Environment Variable | Examples values                                       | Description                                                                |
| :------------------- | ----------------------------------------------------- | :------------------------------------------------------------------------- |
| `CF_CRED_SPEC_FILE`  | '/var/credentials-fetcher/my-credspec.json'           | Path to a credential spec file used as input. (Lease id default: credspec) |
|                      | '/var/credentials-fetcher/my-credspec.json:myLeaseId' | An optional lease id specified after a colon                               |
| `CF_GMSA_OU`         | 'CN=Managed Service Accounts'                         | Component of GMSA distinguished name (see docs/cf_gmsa_ou.md)              |

## Compatibility

Running the Credentials-fetcher outside of Linux distributions is not
supported.

## Contributing

Contributions and feedback are welcome! Proposals and pull requests will be considered and responded to. For more
information, see the [CONTRIBUTING.md](https://github.com/aws/credentials-fetcher/blob/master/CONTRIBUTING.md) file.
If you have a bug/and issue around the behavior of the credentials-fetcher,
please open it here.

Amazon Web Services does not currently provide support for modified copies of this software.

## Security disclosures

If you think you’ve found a potential security issue, please do not post it in the Issues. Instead, please follow the instructions [here](https://aws.amazon.com/security/vulnerability-reporting/) or [email AWS security directly](mailto:aws-security@amazon.com).

## License

The Credentials Fetcher is licensed under the Apache 2.0 License.
See [LICENSE](LICENSE) and [NOTICE](NOTICE) for more information.
