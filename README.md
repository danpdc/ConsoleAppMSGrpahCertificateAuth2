---
services: active-directory
platforms: dotnet
author: danpdc
---
# Authenticating to Azure AD in daemon apps with certificates

![](https://identitydivision.visualstudio.com/_apis/public/build/definitions/a7934fdd-dcde-4492-a406-7fad6ac00e17/30/badge)
![](https://githuborgrepohealth.azurewebsites.net/api/TestBadge?id=3)

In this sample a Windows console application (GetAllUsers) calls the Microsoft Graph API using its app identity. This scenario is useful for situations where headless or unattended job or process needs to run as an application identity, instead of as a user's identity. The application uses the Active Directory Authentication Library (ADAL) to get a token from Azure AD using the OAuth 2.0 client credential flow, where the client credential is a certificate.

This sample is similar to [Daemon-DotNet](https://github.com/Azure-Samples/active-directory-dotnet-daemon), except instead of the daemon using a password as a credential to authenticate with Azure AD, here it uses a certificate.

For more information about how the protocols work in this scenario and other scenarios, see [Authentication Scenarios for Azure AD](http://go.microsoft.com/fwlink/?LinkId=394414) and [Service to service calls using client credentials](https://github.com/Microsoft/azure-docs/blob/master/articles/active-directory/develop/active-directory-protocols-oauth-service-to-service.md)

## How to Run this sample

To run this sample, you will need:
 - Visual Studio 2013 or above (also tested with Visual Studio 2015 and Visual Studio 2017)
 - An Internet connection
 - An Azure Active Directory (Azure AD) tenant. For more information on how to get an Azure AD tenant, please see [How to get an Azure AD tenant](https://azure.microsoft.com/en-us/documentation/articles/active-directory-howto-tenant/)


### Step 1:  Clone or download this repository

You can clone this repository from Visual Studio. Alternatively, from your shell or command line, use:

`git clone https://github.com/Azure-Samples/active-directory-dotnet-daemon-certificate-credential.git`

### Step 2:  Register the sample with your Azure Active Directory tenant and configure the code accordingly

1. Sign in to the [Azure portal](https://portal.azure.com)
2. On the top bar, click on your account and under the Directory list, choose the Active Directory tenant where you wish to register your application.
3. Click on More Services in the left hand nav, and choose Azure Active Directory.
4. Click on App registrations and choose Add.
5. Enter a friendly name for the application, for example 'GetUsers' and select 'Web Application and/or Web API' as the Application Type (even if here we have a console application).
6. Since this application is a daemon and not a web application, it doesn't have a sign-in URL or app ID URI. For these two fields, enter "http://getusers". Click on Create to create the application.
7. While still in the Azure portal, choose your application, click on Settings and choose Properties.
8. Find the Application ID value and copy it to the clipboard.

### Step 3: Create a self signe certificate

To complete this step you will use the New-SelfSignedCertificate Powershell command. You can find more information about the New-SelfSignedCertificat command here.

Open PowerShell and run New-SelfSignedCertificate with the following parameters to create a self-signed certificate in the user certificate store on your computer:

```

$cert=New-SelfSignedCertificate -Subject "CN=GetUsersCert" -CertStoreLocation "Cert:\CurrentUser\My"  -KeyExportPolicy Exportable -KeySpec Signature

```


### Step 4: Add certificate details to Azure AD manifest file

First you would have to generate all the details and store them in a file and in a format that is consumable by Azure AD
To do this, copy and paste the following code in the same PowerShell session where you generated the sefl signed certificate.

```

$bin = $cert.RawData
$base64Value = [System.Convert]::ToBase64String($bin)
$bin = $cert.GetCertHash()
$base64Thumbprint = [System.Convert]::ToBase64String($bin)
$keyid = [System.Guid]::NewGuid().ToString()
$jsonObj = @{customKeyIdentifier=$base64Thumbprint;keyId=$keyid;type="AsymmetricX509Cert";usage="Verify";value=$base64Value}
$keyCredentials=ConvertTo-Json @($jsonObj) | Out-File "keyCredentials.txt"

```

The content of the generated "keyCredentials.txt" file has the following schema:

```

[
    {
        "customKeyIdentifier": "$base64Thumbprint_from_above",
        "keyId": "$keyid_from_above",
        "type": "AsymmetricX509Cert",
        "usage": "Verify",
        "value":  "$base64Value_from_above"
    }
]

```

To associate the certificate credential with the GetUsers app object in Azure AD, you will need to edit the application manifest. In the Azure Management Portal app registration for the GetUsers app click on Manifest. A blade opens enabling you to edit the manifest. You need to replace the value of the keyCredentials property (that is [] if you don't have any certificate credentials yet), with the content of the keyCredential.txt file

To do this replacement in the manifest, you have two options:

Option 1: Edit the manifest in place by clicking Edit, replacing the keyCredentials value, and then clicking Save. Note that if you refresh the web page, the key is displayed with different properties than what you have input. In particular, you can now see the endDate, and stateDate, and the vlaue is shown as null. This is normal.

Option 2: Download the manifest to your computer, edit it with your favorite text editor, save a copy of it, and Upload this copy. You might want to choose this option if you want to keep track of the history of the manifest.

Note that the keyCredentials property is multi-valued, so you may upload multiple certificates for richer key management. In that case copy only the text between the curly brackets.

### Step 5: Configure application

1. Open `app.config'.
2. Find the app key 'ida:Tenant' and replace the value with your AAD tenant name, something **like contoso.onmicrosoft.com**.
3. Find the app key 'ida:ClientId' and replace the value with the Client ID for the GetUsers app registration from the Azure portal.
4. Find the app key `ida:CertName` and replace the value with the subject name of the self-signed certificate you created, e.g. "CN=GetUsersCert".

Now you should be able to run the application

### Step 6: Run the application

Running the application should not be any big deal. However, make sure that you first right click the project and build it, so that all needed packages are installed. 
The console application will initiate with a 3 seconds delay, and then it will query the Microsoft Graph for all users in the tenant. The user information will be displayed in JSON format, so it might look messy. 

## Some useful information

In `Main()` we define the needed variables and then initialize the certificate that will be used to create the certificate credential to be used by ADAL

```
X509Certificate2 cert = null;
            X509Store store = new X509Store(StoreLocation.CurrentUser);
            try
            {
                store.Open(OpenFlags.ReadOnly);
                X509Certificate2Collection certCollection = store.Certificates;
                // Find unexpired certificates.
                X509Certificate2Collection currentCerts = certCollection.Find(X509FindType.FindByTimeValid, DateTime.Now, false);
                // From the collection of unexpired certificates, find the ones with the correct name.
                X509Certificate2Collection signingCert = currentCerts.Find(X509FindType.FindBySubjectDistinguishedName, certName, false);
                if (signingCert.Count == 0)
                {
                    // No matching certificate found.
                    return -1;
                }
                // Return the first certificate in the collection, has the right name and is current.
                cert = signingCert.OfType< X509Certificate2>().OrderByDescending(c => c.NotBefore).First();
            }

```
Then we create the certificate credential: 

```
 certCred = new ClientAssertionCertificate(clientId, cert);

```

In the `GetAllUsers()` method we then request an access token using the certificate credential and then the application sends a request to the Microsoft Graph to retrieve all users. The access token is presented in the request header. 




