# Netlify CMS External GitHub OAuth Client on Google Cloud Functions

[Netlify CMS](https://www.netlifycms.org/) is an open source headless CMS that can be self-hosted, e.g., on GitHub Pages or S3, for little to no cost. When self-hosting, Netlify CMS requires an authentication backend to provide it with an authorized access token for a CMS user's GitHub account with access to the managed site's Git repository.

**This component serves as the external server-side OAuth Client to manage authentication with GitHub and provide the authenticated access token to Netlify CMS.**

This version was based on the [Generic Python](https://github.com/davidejones/netlify-cms-oauth-provider-python) version and inspired by the [Google Apps Engine](https://github.com/signal-noise/netlify-cms-oauth-provider-python-appengine) version. It was rewritten to run as a single Google Cloud Function at ~zero-cost, with reduced authorized scope exposure, security enhancements for CSRF protection, and optional encrypted secret storage -- making it a good fit for community or open source driven projects.

It is written in **Python 3.7**.

Other implementations can be found in the [Netlify CMS documentation](https://www.netlifycms.org/docs/authentication-backends/#external-oauth-clients).

## Prerequisites
Before installing, you should have the following setup and basic familiarity with the Google Cloud Platform:
* [Google Cloud Platform (GCP)](https://cloud.google.com/) account and project
* [Local Python 3 development environment for GCP](https://cloud.google.com/python/setup)

## **Setup Overview**
Using the Cloud Function involves:
1. Cloning this Git repository
2. [Deploying it to a Google Cloud Function](https://cloud.google.com/functions/docs/deploying/), using any of the available methods (from your local machine, Source Control (requires you to Fork this Git repository), or the Cloud Console)
3. Registering the OAuth App under your GitHub account
4. Configuring the Function's environment variables
5. Configuring your Netlify CMS instance

## Deployment
Through this step, you will be simply creating a disabled, unconfigured Function on GCP.

Deploying the component follows [the standard steps for deploying an HTTP triggered Google Cloud Function](https://cloud.google.com/functions/docs/deploying/).

You may use any of the available methods -- from your local machine, from Source Control (requires you to Fork this Git repository and setup as a mirrored Cloud Source Repository), or from the Cloud Console.

If deploying with the `gcloud` command line, here is an example command to use:
```
gcloud functions deploy FUNCTION_NAME --runtime=python37 --entry-point=cloud_run --set-env-vars=[OAUTH_CLIENT_ID=[YOUR ID],OAUTH_CLIENT_SECRET=[YOUR SECRET],FUNCTION_ENABLED=1,REDIRECT_URL=https://[YOUR APP].cloudfunctions.net/cloud_run/callback] --trigger-http
```
With the following arguments explained:
* FUNCTION_NAME: Your choice of a name for the Cloud Function

Take note of the Function's *Name* and *Endpoint URL* for use in later steps. This information may be found in the Cloud Console under the Cloud Function > Trigger section.

The *Endpoint URL* should have the following format:  
```
https://YOUR_REGION-YOUR_PROJECT_ID.cloudfunctions.net/FUNCTION_NAME
```

## GitHub OAuth App Registration
In this step, you will register this component as an OAuth App under the appropriate GitHub account. 

The OAuth App should be registered under the GitHub account with administrative privledges over the Git repository that Netlify CMS will be updating. For example, for a personal repository, you will register under your GitHub account. For an organization's repository, you will register under the organization, using a GitHub account with administrative access in the organization.

With the appropriate GitHub account, follow these instructions, with these configuration guidelines:
* **Homepage URL**:  <*Function Endpoint URL*>
* **Authorization callback URL**:  <*Function Endpoint URL*>`/callback`

Take note of the *Client ID* and *Client Secret* in a secure place.

## Cloud Function Configuration

Configuration is performed with environment variables, by redeploying the Cloud Function or loaded from a .env file. It's recommended to only use a .env file for a local development environment to avoid exposing sensitive configuration variables in a repository.

You may either manage the Environment Variables through the Cloud Console (by editing the Cloud Function) or by re-deploying with the command line, both of which cause a redeployment.

Here is an example `gcloud` deployment command, with the environment variables:
```
gcloud functions deploy FUNCTION_NAME --runtime=python37 --entry-point=cloud_run --set-env-vars=[OAUTH_CLIENT_ID=<YOUR-OAUTH-CLIENT-ID>,OAUTH_CLIENT_SECRET=<YOUR-OAUTH-CLIENT-SECRET>,FUNCTION_ENABLED=1,REDIRECT_URL=<YOUR-CLOUD-FUNCTION-ENDPOINT-URL>/callback] --trigger-http
```

**List of Environment Variables**

|Name|Description|Example Value|
|----|----|----|
|OAUTH_CLIENT_ID|Provided by GitHub upon registering the Function as an OAuth App.|f432a9casdff1e4b79c57|
|OAUTH_CLIENT_SECRET|[*Sensitive*] Provided by GitHub upon registering the Function as an OAuth App. Should be considered equally sensitive as your password.|c0c6k8ew98m0kq4p85tf4z8f84o9w0h360cbqst6|
|FUNCTION_ENABLED|`Boolean` Toggle for whether the function is actively enabled. Allows the Cloud Function to be disabled while still deployed.|1|
|REDIRECT_URL|Authorization Callback URL provided during the GitHub OAuth App registration, i.e., <*Function Endpoint URL*>`/callback`|https://`project-url`.cloudfunctions.net/`function-name`/callback|
|STATE_STORAGE_COLLECTION|[*Optional*] Path to the Firestore Collection designated for temporary state storage (see below).|`collection-name`/oauth_state_storage/states|
|SSL_ENABLED|[*Local Only*] `Boolean` Toggles SSL during OAuth Authentication Flow. Should only be disabled during local development testing.|1|

### State Storage (Optional)
For additional security, you may configure the component to utilize Google Cloud Firestore to temporarily store a user's state during their authentication session. The state is a random value passed between the OAuth Client (this component) and the OAuth Provider (GitHub) as a mechanism to protect against Cross-Site Request Forgery (CSRF) vulnerability.

⚠️ **This feature requires Firestore, which incurs cost!** For most purposes, the cost will be less than $1, as resource utilization is minimal, but it is non-zero and may increase if you have very high user traffic authenticating to Netlify CMS.

To enable this feature, ensure you have an active Cloud Firestore instance (if not, see [Quickstart](https://cloud.google.com/firestore/docs/quickstart-servers)).

Create a collection in your Firestore database as a designated workspace for the state to be stored. Please note states are only stored temporarily during the authentication flow and then deleted, so data will not accummulate there. The collection may be a sub-collection at any hierarchy.

Configure the `STATE_STORAGE_COLLECTION` environment variable with the path to that collection.

## Netlify CMS Configuration
In this final step, you will configure Netlify CMS to use the Function as its authorization backend.

In the Netlify CMS config file, apply the following configuration (`base_url` and `auth` are the specific configuration parameters related to the OAuth provider):
```
backend:
  name: github
  repo: user/repo   # Path to your GitHub repository
  branch: master    # Branch for Netlify CMS to create Pull Requests against
  base_url: <Function Endpoint Base URL>   # Base URL extracted from the Cloud Function Endpoint URL
  auth_endpoint: <Function Name>/auth
```

You will extract parts of the Cloud Function's Endpoint URL into configuration settings in Netlify CMS. The *Endpoint URL* should have the following format:  
```
https://YOUR_REGION-YOUR_PROJECT_ID.cloudfunctions.net/FUNCTION_NAME
```

So, the configuration parameters should be something like:
* `base_url: https://YOUR_REGION-YOUR_PROJECT_ID.cloudfunctions.net`
* `auth_endpoint: FUNCTION_NAME/auth`


Please note that `base_url` should not have trailing slashes.

Your users should now be able to login to your self-hosted Netlify CMS with their GitHub credentials. ✨
