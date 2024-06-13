# Integrate with Azure AD

## [Official Documentation](https://docs.openshift.com/container-platform/4.14/authentication/identity_providers/configuring-oidc-identity-provider.html)

## Process

On the left go to `Administration -> Cluster Settings` and go to **Configuration Tab** and search for **Oauth**.


![azure-oauth](/docs/images/azure-oauth.png)



Click on `OAuth`, scroll to the bottom where it lists **Identity Providers**, drop down `Add` and select `OpenID Connect`:

![azure-oauth](/docs/images/azure-addProvider.png)


Set the name to **azure** and populate the other fields from the `Azure Application Registration`.  The last thing you may need to change is the **preferred_name** mapping.  You may need to get information from the AD Administrator to know what this maps to.  The POC that this was done for had to map this to **premisessamaccountname** within AD to ensure the user names shown properly in Openshift.


![azure-oauth](/docs/images/azure-openIDConnect.png)

**If you make any changes to the Identity Provider, ensure you delete both the user and the identity in Openshift to ensure that there’s no tokens getting stuck anywhere.**
