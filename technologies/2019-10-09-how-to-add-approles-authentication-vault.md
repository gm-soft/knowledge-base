---
category: technologies
tags: [technologies, hashicorp vault, secrets, .net]
---

# Adding an application in Hashicorp Vault

## Enabling AppRole authentication method

First you should enable new Authentication Method in Vault web application.

![add_app_role 1](/technologies/key-vault-assets/add_app_role.PNG)

Then you type name. Let's leave it 'as is'.

![add_app_role 2](/technologies/key-vault-assets/add_app_role_2.PNG)

Then you should generate RoleId and SecretId for your application. This values are some kind of login and password for application which allow the application to get an access token. We may use any string. I will generate GUIDs using [Online GUID Generator](https://www.guidgenerator.com/).

![guid generator page](/technologies/key-vault-assets/guid_generator.PNG)

Then we write this two GUIDs as RoleId and SecretId in our application.

![add_app_role 2](/technologies/key-vault-assets/app_settings.png)

We use this values to get Vault token for our application.

## Setting up a .NET application

To use Vault secrets, you should add a provider for reading Auth Token and also a provider for getting keys.

```csharp

internal class AuthResponse : IAuthResponse
{
    public string Accessor { get; }

    public string ClientToken { get; }

    public bool Renewable { get; }

    public int LeaseDuration { get; }

    [JsonConstructor]
    public AuthResponse(
        [JsonProperty("accessor")] string accessor,
        [JsonProperty("client_token")] string clientToken,
        [JsonProperty("renewable")] bool renewable,
        [JsonProperty("lease_duration")] int leaseDuration)
    {
        Accessor = accessor;
        ClientToken = clientToken;
        Renewable = renewable;
        LeaseDuration = leaseDuration;
    }
}

public async Task<IAuthResponse> GetTokenAsync(CancellationToken cancellationToken)
{
    // Server Uri and LoginPath you may find in your settings.
    var requestUrl = new Uri(new Uri(_config.ServerUri), _config.LoginPath);
    var requestPayload = new
    {
        role_id = _config.AppRoleId, // Role Id (login)
        secret_id = _config.AppSecretId // secret Id (password)
    };

    var request = new HttpRequestMessage(HttpMethod.Post, requestUrl)
    {
        Content = new ObjectContent<dynamic>(requestPayload, new JsonMediaTypeFormatter())
    };

    // If you have namespace, you should add it via http headers
    if (!string.IsNullOrWhiteSpace(_config.Namespace))
    {
        request.Headers.Add("X-Vault-Namespace", _config.Namespace);
    }

    var response = await _httpProvider.SendAsync(request, cancellationToken);
    response.EnsureSuccessStatusCode();

    var responseJson = await response.Content.ReadAsAsync<JObject>(cancellationToken);
    return responseJson["auth"].ToObject<AuthResponse>();
}

```

Now we have a token that allows us to get secrets from the Vault storage.

```csharp

// using section start
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Eds.KeyVault.Contracts.Interfaces;
using VaultSharp;
using VaultSharp.V1.AuthMethods.Token;
// using section end

public class KeyVaultSecretClient : IKeyVaultSecretClient
{
    // client from VaultSharp namespace.
    private readonly IVaultClient _vaultClient;

    public KeyVaultSecretClient(IVaultClient vaultClient)
    {
        _vaultClient = vaultClient ?? throw new ArgumentNullException(nameof(vaultClient));
    }

    public KeyVaultSecretClient(KeyVaultSecretClientConfig config, string clientToken)
    {
        if (config == null)
        {
            throw new ArgumentNullException(nameof(config));
        }

        var settings = new VaultClientSettings(config.ServerUri, new TokenAuthMethodInfo(clientToken));
        if (!string.IsNullOrWhiteSpace(config.Namespace))
        {
            settings.BeforeApiRequestAction = (httpClient, message) => { message.Headers.Add("X-Vault-Namespace", config.Namespace); };
        }

        _vaultClient = new VaultClient(settings);
    }

    public async Task<IDictionary<string, object>> GetSecretsAsync(string secretPath, CancellationToken cancellationToken)
    {
        var secret = await _vaultClient.V1.Secrets.KeyValue.V1.ReadSecretAsync(secretPath);

        return secret.Data;
    }
}

```

Now you may use a dictionary of all key-values from the Vault.

## Add new AppRole auth record in Vault

AN official documentation is available [here](https://www.vaultproject.io/docs/auth/approle.html). We are going to add new AppRole record via Vault console.

![Vault console](/technologies/key-vault-assets/console.PNG)

And then we type next commands:

```console

vault write auth/approle/role/<app_name> role_id=<GUID_for_role_id>
vault write auth/approle/role/<app_name> policies=<policy_name>
vault write auth/approle/role/<app_name>/custom-secret-id secret_id=<secret_id_value>
vault read auth/approle/role/<app_name>

```

Hashicorp Vault generates secret-id automatically by command `vault write -f auth/approle/role/my-role/secret-id`, but we are enable to write our own secret-id value. We have done it above via command `vault write auth/approle/role/<app_name>/custom-secret-id secret_id=<secret_id_value>`.

## Create a new policy

Policy is a file which declares what does your application enable to do: reading, writing or removing secrets. We are going to give readonly permissions. As far as you noticed above, we have added a policy reference via command `vault write auth/approle/role/<app_name> policies=<policy_name>`, and now we have to add this policy.

![Create new policy](/technologies/key-vault-assets/create_policy.PNG)

Then we just type a name of the policy and also type permissions in a field. For example, we want to add readonly permission to all secrets stored in `awesome_namespace` namespace.

```console

# Here is a comment for you path/permission record
path "secret/awesome_namespace*" { capabilities = ["read"] }

```

You may find another one example in official documentation [here](https://learn.hashicorp.com/vault/secrets-management/sm-static-secrets#policy-requirements).

## Add your secret record

First you should enable new secret engine. We are going to create 'Version 1' of the engine, because it is simpler than V2. You may find additional information in documentation [here](https://learn.hashicorp.com/vault/getting-started/secrets-engines).

![enable secret engine](/technologies/key-vault-assets/enable_secret_engine.PNG)

![awesome_name_of_secret_engine](/technologies/key-vault-assets/awesome_name_of_secret_engine.PNG)

Then we should add secret. Here we create a storage file with name `connection_strings` inside `awesome_namespace` namespace.

![created secret](/technologies/key-vault-assets/created_secret.PNG)

Now, we may point our application to this secret connection string.

```json

"ConnectionStrings": {
    "DocumentsDatabase": "vault://awesome_namespace/connection_strings:db"
},

```

We use `vault://` prefix to mark that this string is a vault value. We are going to remove this prefix during runtime parsing process. The prefix is optional, so you don't have to use it.

## Storing files / certificates in Vault

If you have to store certificates in Vault, you may follow next way: you may just store base64 string representation of the file and create the file during your application is starting.

## Useful links

- [Static Secrets: Key/Value Secret Engine](https://learn.hashicorp.com/vault/secrets-management/sm-static-secrets)
