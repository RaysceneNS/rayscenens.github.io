---
title: "How-to use Google Assistant to control AudioStation on your home NAS"
tags: ["Google Assistant", AudioStation, Synology]
---

IMPORTANT! Google has decided to sunset [Conversational Actions on June 13, 2023](https://developers.google.com/assistant/ca-sunset), as a result this configuration in its current form will cease to operate at that time. 

Like many folks who own a Synology NAS I watched with envy as folks in the Alexa universe received love from the development team at Synology to command and control playback of the media that they had placed into the AudioStation app, while those of us within the Google verse got nada. Time to take matters into my own hands. ![google_assistant](/assets/images/2020/12/20/audiostation_command.webp)

 In order to figure out how to make this work I would have to do some digging into how the [Alexa skill communicated with AudioStation](https://racineennis.ca/2020/12/19/AudioStation-GoogleAssistant-investigation).

If you would like to setup your own Google Assistant to Synology AudioStation link then you will need to have setup a DNS name for your NAS and have a valid SSL certificate for that same DNS name as well as have exposed the DSM port (5001 by default) to the internet. Essentially you must be able to login to your DSM from the internet without getting any certificate warnings in your web browser. If you meet these requirements then follow these steps:

## Create a Google Actions project

In order for commands received by your Google assistant to do anything you'll need to setup a Google actions project. Voice assistant commands are connected to a set of server less functions that can communication with the AudioStation service available on our NAS. We will setup our action in test mode only as the connection will be linked to our servers DNS name.

Go to <https://console.actions.google.com/>

Create a new project. You may enter any project name that you like, be sure to select the appropriate language and region for your situation. Now press 'Create Project'.
![new project](/assets/images/2020/12/20/google_console_new_project.png)

Next we will have to select the kind of action that we are building, select 'Custom' and the press 'Next'.
![project type](/assets/images/2020/12/20/google_console_project_type.png)

Now select Blank project, we will be configuring all of the conversation components ourselves. Then press 'Start Building'
![how to build](/assets/images/2020/12/20/google_console_how_to_build.png)

Once the project is created we need to take note of the Project Id, this will be required several times throughout the setup. The project Id can be found by Going to the new project, click the ellipsis, ![settings](/assets/images/2020/12/20/google_console_settings.png)
 select 'Project settings' from the menu. The Project ID is displayed here in the project settings page.
 ![project id](/assets/images/2020/12/20/google_console_project_id.png)

## Register client application with Synology OAuth Service

We will use the existing OAuth service on the Synology NAS. This is the same service that is used by the Alexa skill. Adding a new database entry allows Google assistant to request an authorization token for access to the AudioStation API service using authorization code flow.

From an SSH terminal connected to your NAS use the command `oauth_clientinfo --client-add` to add a client to the OAuth list. You must replace [some-project-id] with the id of the project that we created in the Google actions console earlier.

```SHELL
# ./oauth_clientinfo --client-add oauth-redirect.googleusercontent.com/r/[some-project-id] AudioStation.voiceassistant GoogleAssistant 
```

Important! Make note of the client id and secret returned from calling  oauth_clientinfo as we require these later on to complete the service registration with Google.

Now you should see something like the following in your OAuth service blade.
![Alternate Product](/assets/images/2020/12/20/oauth_setup.webp)

## Install Google Actions CLI

The actions CLI is a Googles command line interface for working with  Actions projects. We will use the CLI to deploy Invocations and our web hook code to the Actions Console project we created earlier.

1. Node.js and NPM
    + recommend installing using [nvm for Linux/Mac](https://github.com/creationix/nvm) and [nvm-windows for Windows](https://github.com/coreybutler/nvm-windows)
1. Install the [Firebase CLI](https://developers.google.com/assistant/conversational/deploy-fulfillment)
    + We recommend using MAJOR version `8`, `npm install -g firebase-tools@^8.0.0`
    + Run `firebase login` with your Google account
1. Install the [Actions CLI](https://developers.google.com/assistant/actionssdk/gactions)

## GitHub Code Deployment

We will now deploy the code to setup our action, this is where we define how spoken commands are translated into requests for media playback.

Create a new directory on your local computer and clone the GitHub repository to it using git

```bash
git clone https://github.com/RaysceneNS/AudioStation-GoogleAssistant.git
```

Navigate to `settings/settings.yaml` and make the following edits:

1. replace '<PROJECT_ID>' with your project ID from Google Console
1. replace '<SERVER_NAME>' with the DNS name of your Synology NAS
1. replace '<CLIENT_ID>' with the client id returned from the OAuth registration we made earlier.

### Add client secret to the project

We must encrypt the client secret that was generated when we registered the OAuth client with Synology. Use the encrypt command to enter the client secret.

```bash
gactions encrypt
Write your secret: *********************
Encrypting your client secret
```

### Deploy the project code

Now that our project is ready we can push the code from our local machine up to the Google cloud for testing.

1. Run `gactions login` to login to your account.
1. Run `gactions push` to push your project.
1. Run `gactions deploy preview` to deploy the project.

After the deployment is successful you should see be able to see the project within the Google Actions Console.

### Environment Variables

One final step is to enter an environment variable that is used to connect the web hook handler code to the external DNS name of your NAS.

1. Go to [https://console.cloud.google.com/](https://console.cloud.google.com/).
1. Select the project that this action is setup within. ![Cloud Project Selection](/assets/images/2020/12/20/cloud_console_project.png)
1. Select the cloud function that is deployed from the Google actions console. ![Console functions listing](/assets/images/2020/12/20/cloud_console_functions.png)
1. Edit the cloud function by click on its name in the list of cloud functions. From the Function details page press Edit. ![Console function](/assets/images/2020/12/20/cloud_console_function.png)
1. Expand RUNTIME, BUILD AND CONNECTIONS SETTINGS. Enter a new Runtime Environment variable named API_HOST and enter the domain name of the server that runs AudioStation i.e. my_server_dns.synology.me ![Variable Entry](/assets/images/2020/12/20/cloud_console_variable.png) *NEW* A second environment variable named API_PORT can be used to specify the port number for communicating to your Synology DSM, by default the port is 5001 - however best practice is to change this to a unique port number.  

## Testing

Once the action and fulfillment are setup, you can begin a conversation with Audio Station using Google. Utterances must explicitly include 'Ask Audio Station' or else Google will take this to mean use the default music provider to satisfy the request. Note that this assumes that you've named your action Audio Station, replace as necessary for your situation.
