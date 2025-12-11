# Using OpenBao

In our project, we use OpenBao to increase security in different namespaces.
OpenBao is a powerful tool for managing secrets, such as passwords, tokens, and encryption keys. It is a community-driven, open-source fork of HashiCorp Vault, created in response to the relicensing of Vault under the Business Source License (BSL). OpenBao maintains compatibility with Vault while remaining under a permissive open-source license.
Additionally, our environment has been enriched with modules that facilitate integration between OpenBao and our application.

Applications retrieve all necessary passwords, keys, and tokens from OpenBao based on the appropriate definitions contained in the application.yaml file.

**_IMPORTANT_**: **dataprovider_namespace**-gitea secret is not in use as Gitea credential yet.

Example entries in OpenBao look like this:

<img src="../images/OpenBao01.png" alt="OpenBao01" width="600"><BR>
<img src="../images/OpenBao02.png" alt="OpenBao02" width="400"><BR>

**_As an update from previous version, most of the OpenBao configuration is now applied automatically.
You just need to create a key for Signer and update a couple of values, which is mentioned in other agents readmes._**

**_All the credentials (for Keycloak and other components) are also now automatically stored in OpenBao - review the secrets for credentials if needed._**

You can access OpenBao on <https://secrets.**namespacetag**.**domainsuffix**>

We can always check the actual address in rancher:

<img src="../images/OpenBao03.png" alt="OpenBao03" width="600"><BR>

Root token can be found in secret secrets-root-token, in key token. 

<img src="../images/OpenBao04.png" alt="OpenBao04" width="600"><BR>
<img src="../images/OpenBao05.png" alt="OpenBao05" width="600"><BR>
<img src="../images/OpenBao06.png" alt="OpenBao06" width="400"><BR>

The application retrieves them according to the following configuration:

![OpenBao07 view](../images/OpenBao07.png)

The above configuration ensures that passwords and other secrets are generated securely and managed efficiently, reducing the risk of security breaches and simplifying the management of secrets across namespaces.


