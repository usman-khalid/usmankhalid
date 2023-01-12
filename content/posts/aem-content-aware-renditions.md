---
title: Creating content aware renditions for AEM assets using Asset Compute Service & Azure Cognitive Services
date: 2023-01-12
description: In this article, we'll go over a step by step approach to extending Adobe's asset compute service and using external APIs to generate smart asset renditions and apply them to assets in the AEM DAM.
tags: ["AEM", "Asset Compute Service", "Azure Cognitive Services", "Adobe"]
---

Asset renditions are an important piece of the DAM in Adobe Experience Manager. Out of the box, AEM creates a few renditions for each asset that is uploaded. These can be used for different viewports and devices or as thumbnails. However, what if we wanted to create a blurred or grayscale rendition for an asset? These are some simple use cases, but with Adobe’s [asset compute service](https://experienceleague.adobe.com/docs/asset-compute/using/introduction.html#:~:text=Asset%20Compute%20Service%20is%20a,text%20and%20metadata%2C%20and%20archives.), we can extend the out of the box functionality to handle them and go even further to implement more complex scenarios.

## Asset Compute Service in a nutshell
Up until AEM as a Cloud Service was released by Adobe, creating asset renditions was a process handled internally by the AEM server via workflows. With the introduction of AEM cloud, this process was moved to an external microservice — the asset compute service. Now, asset binaries are uploaded to blob storage through this service when an asset is added to AEM. This is then processed externally by the microservice asynchronously and sent back instead of being handled by AEM itself, freeing up the AEM server from having to do this internally.

The asset compute service is built on the [Adobe/IO Runtime platform](https://developer.adobe.com/runtime/docs/guides/overview/what_is_runtime/), allowing us to extend and customize it by creating custom workers that can utilize external libraries and create all sorts of different renditions. We can do this by creating an [App Builder](https://developer.adobe.com/app-builder/docs/overview/firefly_and_runtime/) (previously known as Firefly) project that can be integrated with AEM. To understand the architecture of the asset compute service in more depth, see the [Adobe documentation](https://experienceleague.adobe.com/docs/asset-compute/using/architecture.html%3Flang%3Dko).

## The use case
Let’s say we have a client that wants to be able to generate headshots for employee photos without having to crop each photo manually and uploading it to the DAM. Using the asset compute service in conjunction with [Azure Computer Vision Service](https://azure.microsoft.com/en-us/products/cognitive-services/computer-vision/#overview), we can create a custom rendition by analyzing the image and [cropping around the area of interest](https://learn.microsoft.com/en-us/azure/cognitive-services/computer-vision/overview-image-analysis?tabs=3-2#get-the-area-of-interest--smart-crop).

To satisfy this requirement, we’ll create a custom asset compute worker which makes a request to the Computer Vision API and creates a rendition out of the cropped image buffer returned in the response.

We’ll set up an asset compute project utilizing the [asset compute SDK](https://www.npmjs.com/package/@adobe/asset-compute-sdk), set it up for local testing and finally create an asset processing profile to integrate it with AEM.

## Prerequisites
There are a few requirements to get our environment set up. We will need:

* An Adobe account linked to an organization, and the organization must have App Builder enabled
* The adobe account must have the correct access to be able to create App Builder projects and add the required services (depending on how the organization is set up, this may require administrative permissions, but in most cases a Developer account will be enough). If the organization does not have access to App Builder, a trial can be requested [here](https://developer.adobe.com/app-builder/docs/overview/getting_access/)
* An Azure Storage account
* Access to the Azure Computer Vision service

_Note: For testing purposes, the free tiers of the Azure services will work just fine._

## Creating the App Builder project
Let's start by creating an App Builder project. Log in to [Adobe Developer Console](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal).

At the top right of the page, click _Create Project_, then select Project from template. Select the _App Builder_ template.

![Creating the project - Step 1](/images/posts/aem-content-aware-renditions/image1.png)

Give the project a title and a name. Add a Development workspace. Ensure Include Runtime with each workspace is checked. Hit _Save_.

![Creating the project - Step 2](/images/posts/aem-content-aware-renditions/image2.png)

Select the newly created project and in the development workspace, add the _Asset Compute service_.

![Creating the project - Step 3](/images/posts/aem-content-aware-renditions/image3.png)

Generate a key pair and download the zip file. The `private.key` file will be used in our local development workflow later. On the next screen, add the Cloud Service integration.

![Creating the project - Step 4](/images/posts/aem-content-aware-renditions/image4.png)

Hit _Save Configured API_. 

We need to add a couple of other APIs from Adobe IO. Add the following services:

* Adobe Services > I/O Events
* Adobe Services > I/O Management API

When completed, the development workspace should look something like the below screenshot.

![Creating the project - Step 5](/images/posts/aem-content-aware-renditions/image5.png)
## Setting up the Azure services
### Azure Storage Account

To set up the local development environment, we’ll need an Azure Storage account and container:

* Log in to the [Azure Portal](https://portal.azure.com/#home)
* Select Create a Resource and search for Storage Account. A Resource Group may need to be added for this
* Click Review. The wizard validates the data
* Hit Create

![Setting up the azure storage account](/images/posts/aem-content-aware-renditions/image6.png)

We will also need to create a container for the storage account:
* Select the newly created storage account from the portal homepage
* From the menu on the left, Select Containers and add one. Call it `assetcomputestorage-container`

### Computer Vision Service
To add the Computer Vision service:
* Select Create a Resource, then search for Computer Vision
* Select the Resource Group created in the previous step
* Hit Review + Create

![Setting up the computer vision service](/images/posts/aem-content-aware-renditions/image7.png)


## Creating the node project
We now have everything we need to develop our custom worker. Let’s create the node project using the Adobe IO CLI.

```
npm i -g @adobe/aio-cli
```

We can now use the CLI to generate a templated asset compute project. The CLI internally uses a custom yeoman generator to scaffold the project.

```
aio app init asset-compute-custom-worker
```

The above command launches a prompt and goes through a few questions and it requires Adobe authentication. It may open a browser window and ask to login with an Adobe ID.

Once authenticated:
* Select the organization associated with the Adobe account
* Select the App Builder project created earlier (in this case, Asset Compute Custom Worker)
* Install the `@adobe/generator-app-asset-compute` template.

The wizard will generate the project with the required dependencies based on the template chosen. Now, we can begin customizing it to our needs.

Let’s install a couple of additional packages to create the renditions.

[Jimp](https://www.npmjs.com/package/jimp) is an image manipulation library we can use to create and manipulate assets. Additionally, we can use [axios](https://www.npmjs.com/package/axios) to make the HTTP request to the Azure Computer Vision service.

```
npm install axios jimp
```

## Configuring the local development environment
We can now set up our local development environment for testing. The project contains a .env file which will hold all the environment variables for the project, including the Azure storage account and Computer Vision service.

We’ll use the development workspace we created to do our work, but the project defaults to the stage workspace. To switch workspaces and use the correct environment variables to populate a base .env file, we can use an aio command.

```
aio app use
```

From the prompt, select Switch to another Workspace in the current project. Then, select the Development workspace. Overwrite both the `.env` and `.aio` files.

### Add the private key and generate console.json for local development
The local development tool that comes with the project requires a console.json file with all the appropriate credentials to connect to Adobe IO. To get this file:

* Go to the Development workspace for the project created earlier in the Adobe Developer Console.
* From the top right, select Download All.
* Rename the downloaded file to console.json
* Place this file in the root of the project

![Private key and console.json](/images/posts/aem-content-aware-renditions/image8.png)



Additionally, add the absolute path to the project key which was downloaded earlier when creating the App Builder project:

```
ASSET_COMPUTE_PRIVATE_KEY_FILE_PATH=/Users/<username>/credentials/asset-compute-custom-worker/private.key
```


_Note: .env, .aio and console.json must be added to the .gitignore file and not be committed to a repository._

### Add the Azure Storage Account environment variables

We can now add the environment variables required for the Azure storage account. Add the following lines to the `.env` file and populate the values appropriately.

_Note: To get the key for the Azure Storage account, select it in the Azure Portal, then go to Access Keys from the menu on the left. The storage account uses rotating keys — we can the copy the first one and use it in our project._

```
AZURE_STORAGE_ACCOUNT=assetcomputestorage
AZURE_STORAGE_KEY=<AZURE_STORAGE_KEY>
AZURE_STORAGE_CONTAINER_NAME=assetcomputestorage-container
```

### Add environment variables for the Azure Computer Vision service
We’ll need to add the base endpoint and the access key for the Azure Computer Vision API as worker parameters for the application and expose them as environment variables. We can do this by adding them as inputs to our custom worker in the `ext.config.yml file.

First, populate the key and the endpoint in the `.env` file:

_Note: These can be accessed by selecting the service in the Azure Portal and navigating to Keys and Endpoint under Resource Management from the menu on the left._

```
AZURE_COMPUTER_VISION_KEY=<AZURE_COMPUTER_VISION_KEY>
AZURE_COMPUTER_VISION_BASE_ENDPOINT=<AZURE_COMPUTER_VISION_ENDPOINT>
```

Update `asset-compute-custom-worker/src/dx-asset-compute-worker-1/ext.config.yml` with the following. Note that the Azure environment variables have been added under inputs.

```
operations:
 workerProcess:
   - type: action
     impl: dx-asset-compute-worker-1/worker
hooks:
 post-app-run: adobe-asset-compute asset-compute:devtool
 test: adobe-asset-compute asset-compute:test-worker
actions: actions
runtimeManifest:
 packages:
   dx-asset-compute-worker-1:
     license: Apache-2.0
     actions:
       worker:
         function: actions/worker/index.js
         web: 'yes'
         runtime: nodejs:16
         inputs:
           AZURE_COMPUTER_VISION_KEY: ${AZURE_COMPUTER_VISION_KEY}
           AZURE_COMPUTER_VISION_BASE_ENDPOINT: ${AZURE_COMPUTER_VISION_BASE_ENDPOINT}
         limits:
           concurrency: 10
         annotations:
           require-adobe-auth: true
```

### Completed .env file

Our .env file should now look something like this:

```AIO_runtime_auth=450c67c2…<auth-credentials>
AIO_runtime_namespace=57606-<projectname>-development
AIO_runtime_apihost=https://adobeioruntime.net
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_client__id=50ab6020042e424f8f9a7c11c8da4cef
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_client__secret=p8e-irx6-uE2LqLRmC1a_ftOpgVwE_pvUBIw
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_technical__account__email=024c452c-8ec0-427f-92da-ee952661e1e3@techacct.adobe.com
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_technical__account__id=835F567E63A085370A495C6B@techacct.adobe.com
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_meta__scopes=["asset_compute_meta"]
AIO_ims_contexts_Asset__Compute__Custom__Worker__J__1671464247289_ims__org__id=0B96B03459707BE40A495C70@AdobeOrg
SERVICE_API_KEY=50ab6020042e424f8f9a7c11c8da4cef
AZURE_STORAGE_ACCOUNT=assetcomputestorage
AZURE_STORAGE_KEY=dRVAiy43WYdqy…
AZURE_STORAGE_CONTAINER_NAME=assetcomputestorage-container
ASSET_COMPUTE_PRIVATE_KEY_FILE_PATH=/Users/<username>/credentials/asset-compute-custom-worker/private.key
AZURE_COMPUTER_VISION_KEY=<AZURE_COMPUTER_VISION_KEY>
AZURE_COMPUTER_VISION_BASE_ENDPOINT=<AZURE_COMPUTER_VISION_BASE_ENDPOINT>
```

## Developing the custom worker
We can now wire up our custom worker using all of the services that we set up. Let’s open up the file at `src/dx-asset-compute-worker-1/actions/worker/index.js` and replace the code with the following:

```javascript
'use strict';

const { worker, GenericError } = require('@adobe/asset-compute-sdk');
const axios = require('axios');
const fs = require('fs');
const Jimp = require('jimp');

const getAzureSmartThumbnail = async (imageUrl, workerParams, { size }) => {
 const { AZURE_COMPUTER_VISION_KEY, AZURE_COMPUTER_VISION_BASE_ENDPOINT } = workerParams;
 const requestUrl = `${AZURE_COMPUTER_VISION_BASE_ENDPOINT}/vision/v3.2/generateThumbnail?width=${size}&height=${size}&smartCropping=true&model-version=latest`;
 const data = {
   url: imageUrl
 };

 const options = {
   headers: {
     'Content-Type': 'application/json',
     'Ocp-Apim-Subscription-Key': AZURE_COMPUTER_VISION_KEY
   },
   responseType: 'arraybuffer',
   responseEncoding: 'binary'
 };

 try {
   const imageResponse = await axios.post(requestUrl, data, options);
   const imageBuffer = Buffer.from(imageResponse.data, 'base64');
   return imageBuffer;
 } catch (error) {
   throw new GenericError(`The Azure Analyze Image API failed with error: ${error}`);
 }
};

exports.main = worker(async (source, rendition, params) => {
 // Service Parameters.
 const imageSize = parseInt(rendition.instructions.size) || 300;
 const imageStyle = rendition.instructions.style;

 // Get the cropped thumbnail from Azure Cognitive Services Computer Vision API.
 const imageBuffer = await getAzureSmartThumbnail(source.url, params, {
   size: imageSize
 });

 try {
   // Create an empty rendition to be used by Jimp.
   const renditionImage = new Jimp(imageSize, imageSize, 0x0);

   // Create a Jimp image out of the buffer returned in the response.
   const image = await Jimp.read(imageBuffer);

   image.crop(
     image.bitmap.width < image.bitmap.height ? 0 : (image.bitmap.width - image.bitmap.height) / 2,
     image.bitmap.width < image.bitmap.height ? (image.bitmap.height - image.bitmap.width) / 2 : 0,
     image.bitmap.width < image.bitmap.height ? image.bitmap.width : image.bitmap.height,
     image.bitmap.width < image.bitmap.height ? image.bitmap.width : image.bitmap.height
   )
   .scaleToFit(imageSize, imageSize);

   if (imageStyle === 'rounded') {
     image.circle();
   }

   // Place the transformed jimp image on top of the placeholder renditionImage.
   renditionImage.composite(image, 0, 0);
  
   // Create the final rendition image.
   fs.writeFileSync(rendition.path, imageBuffer);
   await renditionImage.writeAsync(rendition.path);
 } catch (error) {
   throw new GenericError(`There was an error generating the rendition: ${error}`);
 }
});
```


### What’s happening here?
We can access the custom worker parameters we added earlier via the params object in the main worker function being exported. The `getAzureSmartThumbnail` function makes a request to the Azure Analyze Image API and returns an image buffer in the response. 

We are also utilizing a couple of service parameters that will allow us to create different variations of the asset based on the input in the AEM processing profile. These can be accessed via the rendition.instructions object. In this case, we can use these to choose the size of the asset which will default to 300x300 pixels, and an optional styling option to create a circular rendition if required.

We are using _jimp_ to read the image buffer returned by the Azure Computer Vision service and then manipulating it further to create the rendition(s).

## Testing Locally
Let’s test out our custom worker. To do this, we can run:

```
aio app run
```

Once the project spins up on localhost and our worker is deployed, we should see something like this:

![Asset Compute on Localhost](/images/posts/aem-content-aware-renditions/image9.png)

We can test locally by uploading an asset and running the custom worker against it to create the renditions. The uploaded asset gets stored in the Azure storage account we created earlier, along with any renditions that are created.

_Tip: [Unsplash](https://unsplash.com/) is a great source for free to use licensed stock images for testing._

Once an asset is uploaded for testing, it should be available and prepopulated any time the app is run subsequently for testing.

Click _Run_. The application will process the asset and produce the rendition. If successful, we should see a `300x300` pixel rendition around the area of interest as determined by the Azure Computer Vision service.

![Renditions on localhost](/images/posts/aem-content-aware-renditions/image10.png)

In this case, the excess space around the person in this photo has been removed to create a 300x300 pixel square rendition.
### Using Service Parameters for local testing

Let’s use some of the custom service parameters we added to take this further. We can create a circular rendition that’s `400x400` pixels along with the default 300px square rendition.

To test this out in our local development tool, replace the JSON input in the app to the following:

```json
{
    "renditions": [
        {
            "worker": "https://57606-<projectname>-development.adobeioruntime.net/api/v1/web/dx-asset-compute-worker-1/worker",
            "name": "rendition.jpg"
        },
        {
            "worker": "https://57606-<projectname>-development.adobeioruntime.net/api/v1/web/dx-asset-compute-worker-1/worker",
            "name": "rendition.jpg",
            "size": 400,
            "style": "rounded"
        }
    ]
}
```


After running the worker again, we should get two renditions.

![Renditions on localhost](/images/posts/aem-content-aware-renditions/image11.png)

The first one is the same one we generated earlier. The second one is 400x400 pixels, and has been cropped to be a circle.

## Integrating with AEM as an asset processing profile
Now that we’ve got our custom worker tested and it’s working as expected, we can add it as an asset processing profile in AEM.

_Note: Asset processing profiles can’t be added using local SDK as of the writing of this article, so this step must be done in a Cloud Manager instance._

Navigate to `Tools > Assets > Processing Profiles` in the admin console on the author instance. From the top right, hit Create.

In the project root, run `aio app deploy` to ensure any updates are deployed successfully. Copy the URL for the custom worker. We can now add it to AEM. 

Let’s add the two configurations we tested locally.

![Adding an AEM Asset Processing profile](/images/posts/aem-content-aware-renditions/image12.png)

Click _Save_.

We can now use this processing profile on any asset in the DAM by doing the following:

* Upload an asset to the DAM, then click on it to view its details
* From the top left, select Reprocess Assets
* In the dialog, select the new processing profile that was just created
* Click _Reprocess_

![Using the custom asseet processing profile in AEM](/images/posts/aem-content-aware-renditions/image13.png)

After a few seconds, refresh the page. Once the renditions are generated, they appear in the left rail along with the default ones created by AEM.

![New renditions are available after reprocessing](/images/posts/aem-content-aware-renditions/image14.png)

Alternatively, the custom processing profile can be applied to assets in a specific folder by default. This can be done by navigating to `Tools > Assets > Processing Profiles`, selecting the profile, and clicking the Apply Profile to Folder(s) button.

![Applying custom asset processing profile by default to a DAM folder](/images/posts/aem-content-aware-renditions/image15.png)

## Conclusions
The Asset Compute Service is a powerful, extendable microservice that allows us to customize the way assets are processed in AEM. With the use of external services and APIs, a lot of manual creative effort can be automated to create renditions for specific use cases in AEM.
