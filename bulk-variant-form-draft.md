# Build a Bulk Order Form for Product Variants using the Storefront API, Catalog API, and React

[screenshot of order form]

Sometimes customers want to order multiple variations of a product. Usually, they will need to add a single item to their cart, return to the product page, select different options, and repeat the process. What if we could make it faster and easier to order several variations of a product in 1 action?

With a bulk order form, we can enable the purchase of multiple product variants simultaneously. In this post, we'll cover building an order form in React, retrieving variants from the server side Catalog API using middleware, and adding multiple variants to the cart at the same time using the Storefront Cart API.

If you're interested in providing a bulk order form for multiple simple products, check out the original bulk order form tutorial here [link to original bulk order form article].

## Setting up your Dev Environment
This app is built using the free BigCommerce Cornerstone theme as a base. You can clone this from the GitHub repo here [https://github.com/bigcommerce/cornerstone] or download the original theme from your BigCommerce store's control panel.

You'll want to make sure you have the Stencil CLI installed. This enables local development, allowing us to see changes in real time as we iterate on our project. You can find instructions to install and set up Stencil here [https://developer.bigcommerce.com/stencil-docs/getting-started/installing-stencil]

## Install Dependencies and Configure webpack
Once you have Stencil installed and Cornerstone downloaded, navigate to the Cornerstone directory in your terminal. Run `npm install` to install the default theme dependencies, then install the app dependencies by running this command:

`npm install —-save react react-dom css-loader style-loader @babel/preset-react`

To support React, we also need to configure webpack to use a loader for JSX files and the custom CSS we'll use in our app. Include this in the module rules array in **webpack.common.js**:

**webpack.common.js**

```
{
 test: /\.jsx$/,
 exclude: /node_modules/,
 use: {
 loader: “babel-loader”,
 options: {
 presets: [“@babel/preset-react”],
 },
 }
},
{
 test: /\.css/,
 loader:[ “style-loader”, “css-loader” ]
}
```

You'll also need to add the following to the resolve object:

**webpack.common.js**


```
extensions: [".js", ".jsx"]
```

## Preparing the Layout Template
The bulk order form is going to be implemented on product pages. To make the form easy to use, we'll create a custom layout template for product pages. This will let merchants enable the bulk order form by selecting it as an option in their store control panel.

BigCommerce themes read custom layout files from the directory `templates/pages/custom`. Create the **custom** directory, then include another folder inside it named **product**. This lets a BigCommerce store detect the custom layout file for product pages.

Create a file named `bulk-variant-form.html` in `templates/pages/custom/product`. Next copy the contents of the default `product.html` file located in `templates/pages` into `bulk-variant-form.html`.

In bulk-variant-form.html, we'll wrap `components/products/product-view ` in a div. Next we'll add a couple more elements to act as the root for React and the toggle for the bulk order form.

```
<div id='standardTemplate'>{{> components/products/product-view schema=true }}</div>
<div id='bulk-variant-form' style="display:none"></div>
<button id='activateBulkForm' class='button button--primary'>Purchase in bulk</button>
```

## Adding React to the Theme
First, let's create a new directory for our React app at `assets/js/bulk-variant-form`. In the `bulk-variant-form` directory create a new file for our parent component called `bulk-variant-form.jsx`.

**bulk-variant-form.jsx**

```
import React, { Component } from 'react';

export default class BulkVariantForm extends Component {
    constructor(props) {
        super(props);

        this.state = {

        }
    }

    render() {     
        return (
            <div>
                Bulk order form
            </div>
        )
    }
}

```

To include React and mount our app on product pages, we'll need to import React and our parent component in `assets/js/app.js`. Include the following after `import Global from './theme/global';`: 

**app.js**

```
import React from 'react';
import ReactDOM from 'react-dom';
import BulkVariantForm from './bulk-variant-form/bulk-variant-form';
```

At the bottom of `app.js`, we'll write a function to initialize our React app:

**app.js**

```
window.initBulkVariantForm = function initBulkVariantForm(id) {
    ReactDOM.render(
        React.createElement(BulkVariantForm, id, null),
        document.getElementById('bulk-variant-form')
    );
};
```

The `initBulkVariantForm` function takes a product ID as an argument. When we invoke the function on the product page, we'll pass in the product ID so that we can retrieve product variant data through middleware. 

Let's switch back to the product layout file at `bulk-variant-form.html`. At the bottom of the file, add the following code:

```
<script>
const productID = {productID: `{{product.id}}`};
window.initBulkVariantForm(productID);

const btn = document.getElementById('activateBulkForm');
const standardForm = document.getElementById('standardTemplate');
const bulkForm = document.getElementById('bulk-variant-form');

btn.addEventListener('click', (e) => {
    e.preventDefault();
    toggleForm(standardForm, bulkForm, btn);
})

function toggleForm(standard, bulk, toggle) {
    if (bulk.style.display == 'none') {
        standard.style.display = 'none';
        bulk.style.display = 'block';
        toggle.textContent = 'Purchase individually';
        return;
    } 
    if (bulk.style.display == 'block') {
        standard.style.display = 'block';
        bulk.style.display = 'none';
        toggle.textContent = 'Purchase in bulk';
        return;
    }
}
</script>
```

Here we are getting the product ID from the Handlebars value. The `productID` we pass as an argument to `initBulkVariantForm` is defined as an object because it needs to be passed as a prop to our parent component. Once the product ID is available in our component, we will pass it to custom middleware.

## Using Middleware to Make Requests to a Server Side API
