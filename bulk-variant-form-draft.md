# Build a Bulk Order Form for Product Variants

[screenshot of order form]

Sometimes customers want to order multiple variations of a product. Usually, they will need to add a single item to their cart, return to the product page, select different options, and repeat the process. What if we could make it faster and easier to order several variations of a product in 1 action?

With a bulk order form, we can enable the purchase of multiple product variants simultaneously. In this post, we'll cover building an order form in React, retrieving variants from the server side Catalog API using middleware, and adding multiple variants to the cart at the same time using the Storefront Cart API.

If you're interested in providing a bulk order form for multiple simple products, check out the original bulk order form tutorial here [link to original bulk order form article].

## Setting up your Dev Environment
This app is built using the free BigCommerce Cornerstone theme as a base. You can clone this from the GitHub repo here [https://github.com/bigcommerce/cornerstone] or download the original theme from your BigCommerce store's control panel.

You'll want to make sure you have the Stencil CLI installed. This enables local development, allowing us to see changes in real time as we iterate on our project. You can find instructions to install and set up Stencil here [https://developer.bigcommerce.com/stencil-docs/getting-started/installing-stencil]

## Installing Dependencies and Configuring webpack
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

**bulk-variant-form.html**

```
// --snip

<button id='activateBulkForm' class='button button--primary' style='display:block;margin: 0 auto;'>Purchase in bulk</button>

<div itemscope itemtype="http://schema.org/Product">
    <div id='standardTemplate'>{{> components/products/product-view schema=true }}</div>
    <div id='bulk-variant-form' style="display:none"></div>

    // --snip

</div>
```

## Styling the Form
First, let's create a new directory for our React app at `assets/js/bulk-variant-form`. In the `bulk-variant-form` directory create a new file named **styles.css**. I've included all the CSS below to get us up and running quickly, which you can modify per your own preferences:

**styles.css**

```
.bulk-variant-row {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    grid-column-gap: 8px;
    padding: 8px 0;
    border-bottom: 1px solid #e5e5e5;
}

.bulk-variant-col {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.bulk-variant-col > p {
    margin: 0;
}

.bulk-variant-col > img {
    max-width: 80px;
}

.bulk-form-field {
    display: grid;
    max-width: 90%;
    margin: 0 auto;
}

.bulk-button-row {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    grid-column-gap: 8px;
    padding: 16px 0;
}

.qtyField {
    max-width: 64px;
    text-align: center;
    display: block;
}

#bulkAddToCart {
    grid-column: 5/5;
    margin: 0;
    height: 42px;
}

input[type=number]::-webkit-inner-spin-button {
    -webkit-appearance: none;
}

#bulk-messaging {
    grid-column: 4/5;
}
```

## Using the Custom Layout Template in Stencil CLI
When the theme is uploaded to a store, a merchant will have the ability to select the custom layout file when they edit a product in the BigCommerce control panel. 


To use the custom layout while developing with Stencil, include this within your `.stencil` file:

```
"customLayouts": {
    "brand": {},
    "category": {},
    "page": {},
    "product": {
      "bulk-variant-form.html": "/your-product-page-url"
    }
  }
```

If you don't see a file named `.stencil` in your theme's root directory, be sure to run the `stencil init` command in your terminal.

## Adding React to the Theme
In the `bulk-variant-form` directory create a new file named `bulk-variant-form.jsx`. To get started, let's include the following code:

**bulk-variant-form.jsx**

```
import React, { Component } from 'react';
import './styles.css';

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

To include React and mount our app on product pages, we'll need to import React and our parent component in `assets/js/app.js`. Include the following in `app.js`: 

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

**bulk-variant-form.html**

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

First we get the product ID from a Handlebars expression. The `productID` variable we pass as an argument to `initBulkVariantForm` is defined as an object because it needs to be passed as a prop to our parent component. Once the product ID is available in our component, we will pass it to custom middleware.

## Using Middleware to Pass Responses from a Server Side API to the Front End
We need to get all of a product's variant IDs, since they are a required property to add variants to the cart using the Storefront API [link to cart api docs]. However, that data isn't completely available on a product page; the standard add to cart flow only needs to identify 1 variant at a time as a shopper chooses options.

Using the BigCommerce Catalog API [link to docs page], we can retrieve all of a product's variants in 1 or (if your product has more than 250 variants) 2 requests. This is where the magic of middleware comes in, enabling us to safely get data restricted to the server side API and sending it to our front end React app.

Our example middleware is going to be an AWS Lambda Function that makes requests to the Catalog API in response to a GET from our React app. Going into detail about the AWS implementation is out of the scope of this article, but the overall work flow should be applicable to your own middleware app or serverless function.

Remember how we pulled the product ID in our custom layout file and passed it to our React app as props? Now we need to pass that to our middleware so it can make a request to the Catalog API. 

In AWS I've set up an endpoint that responds to a GET request from the React app that includes the product ID as a query parameter. The middleware's URL endpoint requested by the front end looks similar to this:

```
https://xxxxxxxxx.execute-api.region.amazonaws.com/default/middleware-endpoint?product_id={product ID}
```

The Lambda Function reads the product ID from the query param and passes it to a Catalog API request structured like this:

```
GET

https://api.bigcommerce.com/stores/hfdehryc/v3/catalog/products/${id}/variants?include_fields=calculated_price,inventory_level,sku,option_values,image_url
```


The `include_fields` parameter lets us specify which variant properties we want returned. We'll use these properties to populate a table of products in our bulk order form.



Let's go over how to use this data in our React app. In `bulk-variant-form.jsx`, we'll set an initial state in the constructor:

**bulk-variant-form.jsx**

```
this.state = {
    variants: [],
    lineItems: [],
    loaded: false,
    message: ''
}
```

Using the `componentDidMount` lifecycle method, we'll make a fetch to our middleware endpoint and populate our state with the results:

**bulk-variant-form.jsx**

```
componentDidMount() {
    fetch(`https://xxxxxxxxx.execute-api.region.amazonaws.com/default/middleware-endpoint?product_id=${this.props.productID}`)
    .then(response => response.json())
    .then(formatted => {
        const lineItems = formatted['data'].map(variant => {
            return {variantId: variant.id, quantity: 0, productId: parseInt(this.props.productID)}
        });

        this.setState({
            variants: formatted['data'],
            lineItems: lineItems,
            loaded: true
        })
    })
    .catch(err => console.log('Bulk form could not load: ', err))
}
```

The variants in state will pass all the data returned from the API to create rows in our bulk order form. The `lineItems` property is structured to match the `lineItems` requirements for adding variants to the cart using the Storefront API. We are also updating the property `loaded` to indicate that the details we need are ready to be displayed to the user.

## Loading Variant Details in the Order Form
Now that we can pull variants into our application state, we can feed the values into our form. First we'll create the basic layout of the form that will be rendered after the API request to get variants is complete. Within the constructor in `bulk-variant-form.jsx`, add the following method:

**bulk-variant-form.jsx**

```
this.renderAfterLoad = () => {
    if (this.state.loaded) {
        return (
            <div>
                <div className='bulk-variant-row'>
                    <div className='bulk-variant-col'><h5>Images</h5></div>
                    <div className='bulk-variant-col'><h5>Values</h5></div>
                    <div className='bulk-variant-col'><h5>Price</h5></div>
                    <div className='bulk-variant-col'><h5>SKU</h5></div>
                    <div className='bulk-variant-col'><h5>Quantity</h5></div>
                </div>
                
                <div className='bulk-button-row'>
                    <div className='bulk-variant-col' id='bulk-messaging'>{this.state.message}</div>
                    <button className='bulk-variant-col button button--primary' id='bulkAddToCart'>Add to Cart</button>
                </div>
            </div>
        )
    }
}
```

As the component is rendered, we want to make sure the data we need is available before displaying anything. In the component `render` method, we'll return the results of `renderAfterLoad`.

```
render() {     
    return (
        <div className='bulk-form-field'>
            {this.renderAfterLoad()}
        </div>
    )
}
```

This will give us the basic layout of the columns, as well as the add to cart button. Next, we need to build a component that will accept all the variants and render a row for each one.

In `assets/js/bulk-variant-form`, let's add a new file named `bulk-variant-rows.jsx`.  This component will take all the variants as props and create a row for every variant.

**bulk-variant-rows.jsx**

```
import React from 'react';

const filterOptionValues = (option_values) => {
    const nameValues = option_values.map((option, index) => {
        return <p key={index}><strong>{option.option_display_name}</strong>: {option.label}</p>
    })
    return nameValues;
}

const bulkVariantRows = (props) => {
    const variants = props.variants;
    const variantRows = variants.map((variant, index) => {
        return (
            <div key={index} className='bulk-variant-row'>
                    <div className='bulk-variant-col'><img src={variant.image_url}/></div>
                    <div className='bulk-variant-col'>{filterOptionValues(variant.option_values)}</div>
                    <div className='bulk-variant-col'>${parseFloat(variant.calculated_price).toPrecision(4)}</div>
                    <div className='bulk-variant-col'>{variant.sku}</div>
                    <div className='bulk-variant-col'>
                        <input 
                          type='number' 
                          min='0' 
                          placeholder='0' 
                          className='qtyField' 
                        />
                    </div>
            </div>
        )
    })

    return variantRows;
}

export default bulkVariantRows;
```

Since a variant's `option_values` property is an array, we map all the option names and values in `filterOptionValues` and show this to the shopper. As an example, if I'm selling shirts, this could show size and color.

Each element representing a column in the form populates with data from the API, except the last column, which has an input field for the quantity.


Now that we have a component that can consume the variant data and return rows for each variant, let's import it into our parent component:

**bulk-variant-form.jsx**

```
import BulkVariantRows from './bulk-variant-rows';
```

Now we add the `BulkVariantRows` component in our `renderAfterLoad` method:

**bulk-variant-form.jsx**

```
<div className='bulk-variant-row'>
    // --snip
</div>

<BulkVariantRows variants={this.state.variants}/>

<div className='bulk-button-row'>
    // --snip
</div>
```

At this point, when you toggle the bulk order form on a product, you should see the variants listed in rows with their image, option values, price, and sku displayed. Now we need to provide the ability to set quantities and add variants to the cart.

## Setting Variant Quantity
We need to update the lineItems in our component state whenever a shopper sets the quantity of any variant. First, create a new method in `bulk-variant-form.jsx` within the constructor:

**bulk-variant-form.jsx**

```
this.changeQty = (variantID, qty) => {
    this.setState({message: ''});
    const lineItems = [...this.state.lineItems];
    lineItems.forEach(item => {
        if (item.variantId == variantID) {
            item.quantity = parseInt(qty);
        }
    })
    
    this.setState({
        lineItems: lineItems
    })
}
```

This takes the variant ID and quantity set in the input field and updates the `lineItems` in our state so that they're ready to be passed to a Storefront API request.

Now we can pass this as a prop to `BulkVariantRows` so that all our variant rows have access to this method:

**bulk-variant-form.jsx**

```
<BulkVariantRows variants={this.state.variants} changeQty={this.changeQty}/>
```

In `bulk-variant-rows.jsx`, we'll update the number `<input>` field so that it calls `changeQty` in response to an onChange event:

**bulk-variant-rows.jsx**

```
<input 
type='number' 
min='0' 
placeholder='0' 
className='qtyField' 
onChange={(e) => props.changeQty(variant.id, e.target.value)}
/>
```

At this point, adjusting the quantity in the form will update the quantity in our state. Now we need to be able to add items to the cart. In `bulk-variant-form.jsx` we'll add our last method in the constructor:

**bulk-variant-form.jsx**

```
this.addToCart = (e) => {
    const lineItems = this.state.lineItems.map(item => {
        if (item.quantity > 0) {
            return item;
        }
    }).filter(item => {
        if (item !== undefined) {
            return item;
        }
    });
    
    if (lineItems.length < 1) {
        this.setState({message: 'Please set the quantity of at least 1 item.'})
    } else {
        e.target.disabled = true;
        this.setState({message: 'Adding items to your cart...'})
        fetch(`/api/storefront/cart`)
        .then(response => response.json())
        .then(cart => {
            if(cart.length > 0) {
                return addToExistingCart(cart[0].id)
            } else {
                return createNewCart()
            }
        })
        .then(() => window.location = '/cart.php')
        .catch(err => console.log(err))
    }

    async function createNewCart() {
        const response = await fetch(`/api/storefront/carts`, {
            credentials: "include",
            method: "POST",
            body: JSON.stringify({ lineItems: lineItems })
        });
        const data = await response.json();
        if (!response.ok) {
            return Promise.reject("There was an issue adding items to your cart. Please try again.")
        } else {
            console.log(data);
        }
    }
    async function addToExistingCart(cart_id) {
        const response = await fetch(`/api/storefront/carts/${cart_id}/items`, {
            credentials: "include",
            method: "POST",
            body: JSON.stringify({ lineItems: lineItems })
        });
        const data = await response.json();
        if (!response.ok) {
            return Promise.reject("There was an issue adding items to your cart. Please try again.")
        } else {
            console.log(data);     
        }
    }
}
```

To prepare the `lineItems` we'll send to the Storefront Cart API, we need to get the ID of all variants with a quantity of at least 1. Using `map` on an array returns a new array with the same number of items, which means any variant with a quantity of 0 is instead replaced with `undefined` in the new array. 

We filter out all the items that are `undefined`, leaving us with a `lineItems` array that only includes a shopper's selected variants.

Depending on if any `lineItems` are present, a message is displayed to the shopper. If there are `lineItems`, the add to cart button is disabled to prevent multiple submissions, and a fetch is made to the API to check if a cart exists or if a cart needs to be created.

Once the request completes successfully, the shopper is redirected to the cart page.

[image of all the stuff working]

## Summary / TL;DR

On product pages using a custom layout template, a React app is activated in the background after the default product view is rendered. After the React app is mounted on the page, a request is made to send the product's ID to an endpoint belonging to middleware.

The middleware makes a request to the BigCommerce Catalog API to retrieve the product's variants. The middleware sends variant data back to the React app.

A bulk order form is generated from the variants. The variant ID and the quantity for each variant are tracked in the component state.

When a shopper adds variants to their cart, the quantity and variant ID are included in the request to the BigCommerce Storefront Cart API. After the request to the Cart API succeeds, a shopper is redirected to the cart page. 

The shopper completes their purchase, marvelling at how quickly they could buy all those items.