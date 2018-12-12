---
title: Manage Azure Service Fabric application secrets on Linux clusters | Microsoft Docs
description: Learn how to secure secret values in a Service Fabric application on Linux clusters.
services: service-fabric
documentationcenter: .net
author: shsha
manager:
editor: ''

ms.assetid: 94a67e45-7094-4fbd-9c88-51f4fc3c523a
ms.service: service-fabric
ms.devlang: dotnet
ms.topic: conceptual
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 12/11/2018
ms.author: shsha

---
# Manage secrets in Service Fabric applications on Linux clusters
This guide walks you through the steps of managing secrets in a Service Fabric application on Linux clusters. Secrets can be any sensitive information, such as storage connection strings, passwords, or other values that should not be handled in plain text. See [Manage application secrets on Windows clusters][manage-secrets-windows-link] for Windows clusters.

## Obtain a data encipherment certificate
A data encipherment certificate is used strictly for encryption and decryption of [parameters][parameters-link] in a service's Settings.xml and [environment variables][environment-variables-link] in a service's ServiceManifest.xml. It is not used for authentication or signing of cipher text. The certificate must meet the following requirements:

* The certificate must contain a private key.
* The certificate must be created for key exchange, exportable to a Personal Information Exchange (.pfx) file.
* The certificate key usage must include Data Encipherment (10), and should not include Server Authentication or Client Authentication.

  For example, the following commands can be used to generate the required certificate using OpenSSL:
  
  ```console
  user@linux:~$ openssl req -newkey rsa:2048 -nodes -keyout TestCert.prv -x509 -days 365 -out TestCert.pem
  user@linux:~$ cat TestCert.prv >> TestCert.pem
  ```

## Install the certificate in your cluster
The certificate must be installed on each node in the cluster under `/var/lib/sfcerts`. The user account under which the service is running (sfuser by default) **should have read access** to the installed certificate (i.e. `/var/lib/sfcerts/TestCert.pem` for the current example).

## Encrypt application secrets
When deploying an application, encrypt secret values with the certificate and inject them into a service's Settings.xml configuration file. The Service Fabric SDK has built-in secret encryption and decryption functions. Secret values can be encrypted at build-time and then decrypted and read programmatically in service code. 

The following snippet can be used to encrypt a secret. This snippet only encrypts the value; it does **not** sign the cipher text. You must use the same encipherment certificate that is installed in your cluster to produce ciphertext for secret values:

```console
user@linux:$ echo "Hello World!" > plaintext.txt
user@linux:$ iconv -f ASCII -t UTF-16LE plaintext.txt -o plaintext_UTF-16.txt
user@linux:$ openssl smime -encrypt -in plaintext_UTF-16.txt -binary -outform der TestCert.pem | base64 > encrypted.txt
```
The resulting base-64 encoded string output to encrypted.txt contains both the secret ciphertext as well as information about the certificate that was used to encrypt it. You can verify its validity by decrypting it with OpenSSL.
```console
user@linux:$ cat encrypted.txt | base64 -d | openssl smime -decrypt -inform der -inkey TestCert.prv
```

The base-64 encoded string can be inserted as an encrypted [parameter][parameters-link] in your service's Settings.xml configuration file with the `IsEncrypted` attribute set to `true`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Settings xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/2011/01/fabric">
  <Section Name="MySettings">
    <Parameter Name="MySecret" IsEncrypted="true" Value="I6jCCAeYCAxgFhBXABFxzAt ... gNBRyeWFXl2VydmjZNwJIM=" />
  </Section>
</Settings>
```
It can also be specified as an encrypted [environment variable][environment-variables-link] in your service's ServiceManifest.xml file with the `Type` attribute set to `Encrypted`:
```xml
<CodePackage Name="Code" Version="1.0.0">
  <EnvironmentVariables>
    <EnvironmentVariable Name="MyEnvVariable" Type="Encrypted" Value="I6jCCAeYCAxgFhBXABFxzAt ... gNBRyeWFXl2VydmjZNwJIM=" />
  </EnvironmentVariables>
</CodePackage>
```

## Decrypt secrets from service code
The APIs for accessing [parameters][parameters-link] and [environment variables][environment-variables-link] allow for easy decrypting of values:

```csharp
// Access decrypted parameters from Settings.xml
var activationContext = FabricRuntime.GetActivationContext();
var configPackage = activationContext.GetConfigurationPackageObject("Config");
bool MySecretIsEncrypted = configPackage.Settings.Sections["MySettings"].Parameters["MySecret"].IsEncrypted;
SecureString MySecretDecryptedValue = configPackage.Settings.Sections["MySettings"].Parameters["MySecret"].DecryptValue();

// Access decrypted environment variables from ServiceManifest.xml
// Note: you do not have to call any explicit API to decrypt the environment variable.
string MyEnvVariable = Environment.GetEnvironmentVariable("MyEnvVariable");
```

## Next Steps
Learn more about [application and service security](service-fabric-application-and-service-security.md)

<!-- Links -->
[parameters-link]:service-fabric-how-to-parameterize-configuration-files.md
[environment-variables-link]: service-fabric-how-to-specify-environment-variables.md
[manage-secrets-windows-link]: service-fabric-application-secret-management.md
