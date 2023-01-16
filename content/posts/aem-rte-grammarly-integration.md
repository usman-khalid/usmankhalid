---
title: Integrating a typing assistant with AEM to provide grammar suggestions during content creation
date: 2023-01-16
description: In this post, we'll go over a simple integration with Grammarly for an AEM Rich Text Editor (RTE) component, giving authors the ability to check their content for spelling and grammar mistakes.
tags: ["AEM", "RTE", "Granite", "Authoring"]
---

[Grammarly](https://www.grammarly.com/grammar-check) is a service that hooks up to an `input` or `textarea` on a page and provides suggestions on the content's grammar. This makes for a neat little integration with the AEM Text/RTE component. This can be added on to the [WCM Core Text component](https://experienceleague.adobe.com/docs/experience-manager-core-components/using/wcm-components/text.html) or any component that uses an RTE.

## Create the editor clientlib
First thing we need is a clientlib we can load into the AEM page editor.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:ClientLibraryFolder"
    categories="[editor.rte.rteGrammarly]"/>
```

In the clientlib directory, add a a `rteGrammarly.js` and a `rteGrammarly.less` file.

## Initializing Grammarly on dialog ready

The JS code will look like something like this:

```javascript
(($, $document) => {
    const GRAMMARLY_CLIENT_ID = 'YOUR_GRAMMARLY_CLIENT_ID';

    $document.on('dialog-ready', async () => {
        const rteElement = document.querySelector('.cq-RichText-editable');

        /* eslint-disable-next-line */
        const grammarly = await Grammarly.init(GRAMMARLY_CLIENT_ID);

        grammarly.addPlugin(
            rteElement,
            { documentDialect: 'british' },
        );
    });
})($, $(document));
```

We target the `cq-RichText-editable` class which has the `contenteditable` attibrute set on it. This works well with the Grammarly SDK.

Replace the client ID with your Grammarly account client ID. We'll also need to add the library itself. This can be done either via a `script` tag into the AEM template `head`. Alternatively, we can simply include the minified version of the Grammarly SDK into our editor clientlib.


## CSS Customizations
One of the tricky parts is that the Grammarly widget is loaded in a scoped `shadow-root` element, which means we can't simply add CSS for the the classes in our editor clientlib CSS. We can, however, override the CSS variables it uses, which should be good enough for our purposes.

In the `rteGrammarly.less` file — we can customize it like so:

```css
grammarly-editor-plugin {
    --grammarly-button-position-bottom: static;
    --grammarly-button-position-right: 50px;
}
```

This will ensure that the Grammarly widget button will display right underneath the textarea for the RTE instead of being positioned absolutely relative to the dialog.

## Add the clientlib to the component dialog

In our text component, we now just need to add our new clientlib as an `extraClientlib`:

```xml
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:granite="http://www.adobe.com/jcr/granite/1.0"
    jcr:primaryType="nt:unstructured"
    jcr:title="Text"
    sling:resourceType="cq/gui/components/authoring/dialog"
    extraClientlibs="[editor.rte.rteGrammarly]">
```

## Wrapping up

If everything went well, it should work something like this:

![AEM RTE Grammarly Integration Demo](/images/posts/aem-rte-grammarly-integration/aem-grammarly-rte.gif)

## Conclusions

This is a relatively simple integration that can be very useful for content writers and authors. Grammarly also has a premium and business version, which includes more features dealing with content tone and full sentence rewriting, for example. Check out [grammarly.com](https://grammarly.com) for more info.