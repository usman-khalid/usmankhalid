---
title: AEM Franklin — creating a simple custom accordion block
date: 2023-04-11
description: In this post, we'll go over the process to create a simple accordion block in Franklin.
tags: ["AEM", "Franklin", "Helix"]
---

## What is Franklin?

[AEM Franklin](https://www.hlx.live/home) is a new product by Adobe, still in an early beta stage. It's marketed as a lightweight framework, providing the ability to create sites that are fast, quick to set up with an interesting and easy way to author content via Google docs.

AEM Franklin uses the concept of _blocks_ which is the equivalent of components in this lightweight framework. A block is simply a reusable piece that can be used across the website on different pages.

From the [FAQs](https://www.hlx.live/docs/faq):

> A block is a reusable part of a page that is rendered differently than standard content. By default, Franklin will render the content of a document as is. If you need specific functionality and/or styling for certain content, you can place it inside a table with a header row containing the name of the block. Which blocks are available and how to structure the content inside blocks differ from project to project. Many projects have a block inventory ("kitchen sink") page that showcases its available blocks.

## Creating a custom block

Let's create a custom Accordion block that we can use to create interactive content across a website.

### Authoring the content

Let's set up the structure of our content. It may look something like this:

![AEM Franklin sample accordion content](/images/posts/aem-franklin-simple-custom-accordion-block/franklin-accordion-sample-content.png)

With our content deployed, the HTML markup looks something like this:

![AEM Franklin initial markup](/images/posts/aem-franklin-simple-custom-accordion-block/franklin-accordion-initial-markup.png)

### Customizing the block

First, let's create a utility function under `utils/index.js`. This function will help us replace the HTML tags for the accordion triggers to buttons.

```js
/**
 *
 * @param {HTMLElement} oldElement
 * @param {String} newTagName
 * @param {Array<String>} newElementClassNames
 */
export const replaceElementTagName = (oldElement, newTagName, newElementClassNames) => {
  const newElement = document.createElement(newTagName);
  newElement.innerHTML = oldElement.innerHTML;
  oldElement.parentNode.replaceChild(newElement, oldElement);

  if (newElementClassNames) {
    newElement.classList.add(...newElementClassNames);
  }
};

export default {
  replaceElementTagName,
};
```

Create an `accordion` directory under `blocks` and create two files: `accordion.js` and `accordion.css`.

Let's place the following code in `accordion.js`:

```js
import { replaceElementTagName } from '../../utils/index.js';

export default function decorate(block) {
  [...block.children].forEach((accordionItem) => {
    accordionItem.classList.add('accordion-item');

    [...accordionItem.children].forEach((item, index) => {
      if (index === 0) {
        replaceElementTagName(item, 'button', ['accordion-item__trigger']);
      } else {
        item.classList.add('accordion-item__content');
      }
    });
  });

  const accordionTriggers = block.querySelectorAll('.accordion-item__trigger');

  [...accordionTriggers].forEach((trigger) => {
    trigger.addEventListener('click', () => {
      trigger.parentElement.classList.toggle('accordion-item--active');
    });
  });
}

```

Here, we use our utility function to replace the tags for the triggers to buttons. Additionally, we are setting up the event listener on click to toggle the accordion content for each item.

### Adding CSS

In `accordion.css`, add the following to give the block some simple styling:

```css
.accordion-item__trigger {
    border-radius: 0;
    font-size: var(--body-font-size-s);
    margin: 6px 0;
    padding-left: 12px;
    text-align: left;
    width: 100%;
}

.accordion-item__content {
    display: none;
    font-size: var(--body-font-size-xs);
}

.accordion-item--active .accordion-item__content{
    display: flex;
}

.accordion-item--active .accordion-item__trigger {
    background-color: var(--link-hover-color);
}
```

### Wrapping up

If everything went well, our accordion should function like this:

![AEM Franklin custom block demo](/images/posts/aem-franklin-simple-custom-accordion-block/franklin-accordion-demo.gif)

Inspecting the code again, we can see that the structure is a lot clearer now:

![AEM Franklin updated markup](/images/posts/aem-franklin-simple-custom-accordion-block/franklin-accordion-updated-markup.png)
