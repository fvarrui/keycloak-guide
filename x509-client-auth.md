# X.509 client authentication

To enable X.509 client authentication in the KC auth server, follow the next steps:

## Setting up the configuration file

This configuration should be applied before running KC in the `${kc.home.dir}/conf/keycloak.conf` properties file:

### Setting server certificate

In order to enable HTTPS connections you have to set a keypair in the server side: 

```properties
# Path to keystore file (containing a SSL Server certificate signed with CA certificate)
https-key-store-file=${kc.home.dir}/conf/server.jks

# Keystore password
https-key-store-password=*****
```

> Keystore and server certificate password have to be the same.

> Keystore password is in plain text, so it's a good idea to store it in a [vault](https://github.com/keycloak/keycloak-community/blob/main/design/secure-credentials-store.md).

### Enabling mutual TLS (mTLS)

KC also needs a truststore to store certificates to verify clients, required to enable X.509 client authentication:

```properties
# Enabling mutual TLS
https-client-auth=request

# Path to trust-store file (containing the CA certificate)
https-trust-store-file=${kc.home.dir}/conf/softquadrat.jks

# Trust-store password (is in plain text, so it's a good idea to store it in a vault)
https-trust-store-password=*****
```

In our case, the truststore only contains the CA certificate, because it is the root of all the certificates that we want to consider as valid. It is also possible to add each and every one of the client certificates that we want to validate (e.g., if clients use self-signed certificates), but it's not what we want.

## Administration console

After you made changes in the configuration file and run KC, then you have to manage it by accessing through a browser:

![Administration Console](assets/administration-console.png)

Then click on **Administration Console**:

![Login](assets/login.png)

Enter administrator credentials (these credentials are set the first time KC is run) and you will be logged in:

![Logged in](assets/logged-in.png)

#### Create a realm

On the top-left corner, click on **Master > Add realm**:

![Create realm](assets/create-realm.png)

Enter a name for the new realm (e.g.: `datasqill`, better in lowercase) and click on **Create**:

![Set new realm name](assets/set-new-realm-name.png)

> It is possible to import all realm configuration from a JSON file with the **Import** option.

![Realm created](assets/realm-created.png)

And now we have a new realm called `datasqill`. The default configuration in this tab is valid for our purposes. As you can see, by default our new realm supports OpenID Connect and SAML, despite we need only the first one.

#### Set private key for signing JWTs

Go to **Realm Settings > Keys**:

![Realm keys](assets/realm-keys.png)

Click on **Providers** option under **Keys** tab, and choose `java-keystore"` on the **Add keystore...** dropdown list:

![Add keystore](assets/add-keystore.png)

And then, fill the form as follows an click on **Save**:

![Fill in new keystore data](assets/fill-new-keystore-data.png) 

- **Priority**: if there is more than one private key for signing tokens, it will choose the valid (not expired) key with the highest priority (e.g.: 200, to make it higher than KC's default keys).

- **Algorithm**: alg. used when signing (e.g. RSA256)

- **Keystore**: full path to the JKS file.

- **Keystore Password**: password needed to open the keystore.

- **Key Alias**: alias of the key inside the keystore that we want to use for signing.

- **Key Password**: password of the key.

- **Key use**: "sig" for singing tokens ("enc" for encryption).

Then, we can see our new siging key active and in first position:

![New signing key](assets/new-siging-key.png)

#### Set up token validity

To configure the validity time and when a token expires, we must go to **Realm Settings > Tokens** tab:

![Set up issued tokens](assets/setup-issued-tokens.png)

>  By default, access tokens are valid for 5 minutes.

#### Create a client

Clients are the applications which want to log users in to get an access token and we must configure at least one to be able to authenticate and authorize users.

Go to **Clients** section on the left panel and click on **Create** under **Lookup** tab:

![Create client](assets/create-client.png)

Enter a **Client ID** (e.g. `datasqill-client`, which is needed when the client application request an access token), set `openid-connect` as **Client Protocol**, and finally click on **Save**:

![Set new client ID](assets/set-new-client-id.png)

Then, in the **Settings** tab, we have to set a **Description** (e.g. `Datasqill Client Application`), set **Access Type** to `public` (clients do not require a secret), disable **Standard Flow** (not necessary for us, as we only need direct access grants; this flow is for showing a login form to users and then redirect them to an specific URL after a successful login), enable **Direct Access Grants** (the clients already have the users' certificates, that is, their credentials, so it is not necessary to request them) and click on **Save** again:

![New client settings](assets/new-client-settings.png)

##### Disable Refresh Tokens

It is possible to disable the use of a **Refresh Token** when setting up the client. Refresh tokens can be used by clients to request a new access token without the user intervention. 

> In our case, as we are using direct access grants (clients always know user credentials, as they are user certificates), don't need refresh tokens.

Go to the section **OpenID Connect Compatibility Modes** and disable **Use refresh Tokens**:

![Disable refresh tokens](assets/disable-refresh-tokens.png)

#### Add a flow for X.509 certificate authentication

Flows are sequences of steps to authenticate users in KC (or achieve any action related with authentication), and we can create our own custom flows. They can be applied to different user interactions with KC (connecting via a web browser, resetting credentials, direct access to its REST API, ...).

We want that our **users can be authenticated using their X.509 certificates**, so, let's create a new flow for this.

Inside our **Realm**, click on **Authentication** on the left panel, and click on **New** to create a new flow from scratch:

![](assets/new-authentication-flow.png)

Set an **Alias** for the new flow (e.g. `X509`) and click on **Save**:

![](assets/set-new-authflow-data.png)

Then click on **Add execution**:

![](assets/new-authflow-execution.png)

As a **Provider** we must choose `X509/Validate Username` in the dropdown list, and click on **Save** again:

![](assets/set-authflow-provider-x509.png)

After creating the flow execution, we have to configure it expanding **Actions** and choosing **Config**:

![](assets/config-authflow.png)

As mutual TLS is enabled, KC server got the user's certificate when the connection is established. If the connection could be established, it means that the user has a valid certificate (it was trusted with the CA certificate which is stored in the previously configured truststore).

So, now, we have to specify how to extract the user identity from this certificate.

In our case, we want to extract the user ID from the Subject DN using a regular expression, so we choose `Match SubjectDN using a regular expression`. 

Then, set  **User Identity Source** with `UID=(.*?)(?:$)`. In a regular expression, parenthesis defines groups, and they are used to specify which part of the Subject DN will be extracted.

Finally, in **User Mapping Method** we set `Username or Email`, in order to use these fields to find the user in the users database (so, the extracted user identifier from the Subject DN must match with the `username` or `email` attributes of a user in the KC users database). 

Finally, click **Save** at the bottom of the form.

![](assets/setup-x509-authflow.png)

Now, we have to use our new custom flow to enable direct grant access to the KC server, so valid users can request access tokens to KC through its REST API. So, go to **Authentication > Bindings** and choose `X509` as **Direct Grant Flow**, and don't forgot to click on **Save**:

![](assets/direct-grant-flow-x509.png)

Remember that your users have to exist in the KC user database or in any of the federated (Active Directory, LDAP) or custom (Service Provider Interface module) user identity providers.

# References

- [Adding X.509 client certificate authentication to a Direct Grant Flow](https://www.keycloak.org/docs/latest/server_admin/#adding-x-509-client-certificate-authentication-to-a-direct-grant-flow)

- [Configuring TLS](https://www.keycloak.org/server/enabletls)

- [Keycloak with HTTPS & mutual TLS / X.509 authentication | Niko Köbler (@dasniko)](https://www.youtube.com/watch?v=yq1hzNs1JQU)
