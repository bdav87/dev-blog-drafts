# Build a bulk order form using the BigCommerce Storefront API and React.js

[screenshot of order form]

A bulk order form can help your customers check out faster. Normally, a shopper wanting to add many unique items to their cart will have to go back and forth between multiple pages.

Using a bulk order form, a shopper can set the quantity of multiple items and add them all to their cart with the press of a single button. 

The bulk order form we're going to build uses the free BigCommerce Cornerstone theme, the BigCommerce Storefront API, and React.js. It's implemented by setting a custom layout file on a category page with simple products, feeding the product data into the form automatically.

Before you get started, make sure you have the Stencil CLI installed. You can find instructions and download links here: 
https://developer.bigcommerce.com/stencil-docs/getting-started/installing-stencil

Using the CLI will enable running the theme locally, helping us iterate over our changes and viewing them in real time.

We're going to use the latest Cornerstone theme as a base. You can pull the theme from the GitHub repo here: https://github.com/bigcommerce/cornerstone

Once you've got the Stencil CLI installed and the Cornerstone theme downloaded, navigate to the theme directory in your terminal and install these dependencies:

`npm install --save react react-dom css-loader style-loader babel-preset-react`

Since we're introducing React into the theme, we'll need to configure Webpack to use a loader for JSX files. We'll also include a loader for CSS so we can introduce our own styling to the bulk order form. Include this in the module rules array in webpack.common.js:

**webpack.common.js**

```
{
    test: /\.jsx$/,
    exclude: /node_modules/,
    use: {
        loader: "babel-loader",
        options: {
            presets: ["react"],
        },
    }
},
{
    test: /\.css/,
    loader:[ "style-loader", "css-loader" ]
} 
```
You will also need to add the following to the resolve object:

**webpack.common.js**

```
extensions: [".js", ".jsx"]
```

To enable the bulk order form on a category page, we should make it as simple as possible for a merchant to toggle it. Once our app is complete, a merchant should be able to edit a category in their control panel and simply select the bulk order form template.

In BigCommerce themes, custom layout files need to be added to `templates/pages/custom`. Create the `custom` directory, then add another folder inside it named `category`. This lets the store know there is a custom layout file available for category pages.

Create a file named `bulk-order-form.html` in `templates/pages/custom/category`. Next let's copy the contents from the existing `category.html` template to use as a base for our custom template. We don't need any product filtering or side navigation, so let’s remove the following code:

**bulk-order-form.html**
```
{{#if category.faceted_search_enabled}}
    <aside class="page-sidebar" id="faceted-search-container">
        {{> components/category/sidebar}}
    </aside>
    {{else if category.subcategories}}
        <aside class="page-sidebar" id="faceted-search-container">
            {{> components/category/sidebar}}
        </aside>
    {{else if category.shop_by_price}}
        {{#if theme_settings.shop_by_price_visibility}}
             <aside class="page-sidebar" id="faceted-search-container">
                {{> components/category/sidebar}}
            </aside>
        {{/if}}
{{/if}}
```
Since we're replacing the product content normally displayed on the category page, let's also go ahead and clear out everything in between the `<main>` elements:

**bulk-order-form.html**
```
{{#if category.products}}
    {{> components/category/product-listing settings=../settings}}
{{else}}
    <p>{{lang "categories.no_products"}}</p>
{{/if}}
```
React needs an element to build on, so we'll create a div with the ID `bulk-order-form` inside the `<main>` element:

**bulk-order-form.html**
```
<div class="page">
    <main class="page-content" id="product-listing-container">
        <div id="bulk-order-form"></div>
    </main>
</div>
```
Now we'll create our React app directory within the theme files. Create a folder named `bulk-order-form` in `assets/js`. Next create a file inside the `bulk-order-form` directory named `bulk-order-form.jsx`. This will serve as the parent component for the order form.

Ultimately we want to use the product data that's already provided for the page, so we don't have to manually write in product IDs or include direct image links. Our component will use an array for products in the component state that will be updated once product data is fed in as props.

**bulk-order-form.jsx**
```
import React, { Component } from "react";

export default class BulkOrderForm extends Component {
    constructor(props) {
        super(props);

        this.state = {
            products: []
        }
    }
    render() {
        return <div>Bulk order form placeholder</div>
    }
}
```
We have some of the basic foundation set up to really begin working on our app, but there still isn't anything to show for it when running the theme locally in the Stencil CLI and viewing category pages in the browser.

To get our React app appearing on the store, we have to create a function to initialize it in `assets/js/app.js`.

First, let's import React, ReactDOM, and the BulkOrderForm component itself:

**app.js**
```
import React from "react";
import ReactDOM from "react-dom";
import BulkOrderForm from "./bulk-order-form/bulk-order-form";

At the bottom of the file we’ll write our function:
app.js

window.initBulkOrderForm = function initBulkOrderForm(productData) {
    ReactDOM.render(
        React.createElement(BulkOrderForm, productData, null),
        document.getElementById("bulk-order-form")
    );
};
```

The `productData` parameter will be passed into the `BulkOrderForm` component as props. When we call this function in our custom template file, we’ll want to feed in the category products. Let’s swap back to the layout file.

We need to retrieve the product data in a way that's accessible to our JavaScript. Normally the category template uses Handlebars helpers to process the product data and feed it into components like the product cards. However, we need a way to get that data into our React app, outside of the Handlebars based template file. To do so, we can combine BigCommerce’s custom `inject` and `pluck` helpers.

More about inject and pluck helpers:

https://developer.bigcommerce.com/stencil-docs/handlebars-syntax-and-helpers/handlebars-helpers-reference/injection-helpers/injection-helpers#handlebars_inject-and-jscontext

https://developer.bigcommerce.com/stencil-docs/handlebars-syntax-and-helpers/handlebars-helpers-reference/array-helpers/custom-array-helpers#handlebars_pluck


Add the following at the end of the file, after `{{> layout/base}}`

**bulk-order-form.html**

```
{{inject "productIds" (pluck category.products "id")}}
{{inject "productNames" (pluck category.products "name")}}
{{inject "productImages" (pluck category.products "image.data")}}
{{inject "productPrices" (pluck category.products "price.without_tax.formatted")}}

<script>
const pageContext = JSON.parse({{jsContext}});
let productData = [];
pageContext["productIds"].forEach((id, index) => {
    productData.push({
        id: id,
        name: pageContext["productNames"][index],
        image: pageContext["productImages"][index].replace("{:size}", "100x100"),
        price: pageContext["productPrices"][index],
        quantity: 0
    })
});

window.initBulkOrderForm(productData)
</script>
```

The `inject` helper will create a property on the `jsContext` object with the values we ask for in the `pluck` helper. In this case we are grabbing the product id, product name, product image, and product price. We also need to pass in a default quantity of 0 per product.

The product data we’ll feed to the bulk order form props is built after looping through the product IDs in the `jsContext`. All the product properties are injected into `jsContext` in the same order, so we can grab those within the same iteration.

Notice the string replace on `productImages`? This is due to images being returned with `{:size}` included in the image path. Normally this populates with size values specified by the theme configuration when images are fed to the `getImage` helper, but in our situation it just breaks the image URL. Since we want to include several products in a bulk order form, we pass 100x100 into the image URL, which automatically returns a copy of the product image at 100x100 pixels. You can also adjust this to your preference.

We're real close to seeing our React app on the theme for the first time. Now we just need to update our `.stencil` file in the theme directory so that the CLI knows to use our custom layout file for a category.

If you don't see a `.stencil` file in your theme directory, be sure to run the `stencil init` command in your terminal. This generates a file named `.stencil` that includes your store connection information. The BigCommerce developer documentation covers this in detail: 
https://developer.bigcommerce.com/stencil-docs/template-files/custom-templates/authoring-testing-uploading-custom-templates#authoring-testing-uploading_local-mapping


Include the following in your `.stencil` file:

**.stencil**
```
"customLayouts": {
    "brand": {},
    "category": {
      "bulk-order-form.html": "/url-of-your-category-with-simple-products"
    },
    "page": {},
    "product": {}
  }
```
At this point, running the `stencil start` command should show the React app running on the category page you selected.

[screenshot of bulk order form placeholder]

We've confirmed that our React app is successfully injected into the page! Let's get some products loaded in.

We're already asking for the product data and passing it as props in the `initBulkOrderForm` function inside our layout file `bulk-order-form.html`. Now we need to use these to update the products array within our component state in `bulk-order-form.jsx`.

Let's take care of that using the React lifecycle method `componentDidMount`. Add the following to `bulk-order-form.jsx`:

**bulk-order-form.jsx**
```
componentDidMount() {
        let keys = Object.keys(this.props);
        let categoryProducts = []
        keys.forEach(key => {
            key != "children" ? categoryProducts.push(this.props[key]) : null
        });
        this.setState({products: categoryProducts})
}
```
When the product data is passed in as props, this becomes an object with a children property, which we don't need in our component state. This code iterates over all the keys and only pushes the actual product data into a new array, which we then set in the state after the component mounts.

You can see the state update if you include a `console.log` in the render method.

**bulk-order-form.jsx**
```
render() {
        console.log(this.state)
        return <div>Bulk order form placeholder</div>
    }
```

[screenshot of state populated in browser console]

So far, so good. Now we need to turn this data into visual content visible on the page. We're going to create a new component to accept the product data and help render it to the page.

Create a new file in the `assets/js/bulk-order-form` directory named `product-group.jsx`.
Our bulk order form is going to have a table like structure, with columns for the product image, name, price, and quantity. The product state will be managed by the parent component, so we can make `ProductGroup` a stateless functional component.

**product-group.jsx**
```
import React from "react";

const ProductGroup = (props) => {
    return (
        <div className="bulk-form-field">
            <div className="bulk-form-row">
                <div className="bulk-form-col"></div>
                <div className="bulk-form-col"><strong>Product</strong></div>
                <div className="bulk-form-col"><strong>Price</strong></div>
                <div className="bulk-form-col"><strong>Quantity</strong></div>
            </div>
            
            <div className="bulk-form-row">
                <div className="bulk-form-col message"></div>
                <button className="button button--primary bulk-add-cart">Add to Cart</button>
            </div>
        </div>
    )
}

export default ProductGroup;
```
To get our columns and rows organized, we'll use the CSS grid system and add some light styling. Create a file in `assets/js/bulk-order-form` named `bulk-order-form.css`. We'll include the following styles, which you can of course modify to your preferences:

**bulk-order-form.css**
```
#bulk-order-form {
    margin: 0 auto;
    width: 70%;
}

.bulk-form-field {
    display: grid;
    text-align: center;
}

.bulk-form-row {
    display: grid;
    grid-template-columns: 100px repeat(3, 1fr);
    grid-column-gap: 8px;
}

.bulk-form-col {
    padding: 16px;
    display: flex;
    flex-direction: column;
    justify-content: center;
}

.bulk-product-image {
    max-height: 100px;
    width: auto;
    box-shadow: 1px 2px 1px rgba(0, 0, 0, 0.25);
}

.bulk-add-cart {
    width: 180px;
    margin: 0 auto;
    grid-column: 4;
}

.message {
    position: absolute;
    padding: 0;
    margin: 10px;
}
```

Now we can import the stylesheet and `ProductGroup` component in `bulk-order-form.jsx`:

**bulk-order-form.jsx**
```
import ProductGroup from "./product-group";
import "./bulk-order-form.css";
```

In the render method in `bulk-order-form.jsx` we’ll take out our placeholder and pass in the `ProductGroup` component:

**bulk-order-form.jsx**
```
render() {
    return (
        <div>
            <ProductGroup />
        </div>
    )
}
```
[screenshot of column headings and add to cart button]

Now you should see an add to cart button, with 3 columns headers for product, price, and quantity.

Next we need to feed the product data into the `ProductGroup` component, and map the products to individual rows.

First, we'll pass the product data in state as props to `ProductGroup`:

**bulk-order-form.jsx**
```
render() {
    return (
        <div>
            <ProductGroup 
              products={this.state.products}
            />
        </div>
    )
}
```

With the product data passed as props to `ProductGroup`, we now need to map that data on to multiple components representing the individual product rows of the form.

We're going to need a new component, so create a new file in `assets/js/bulk-order-form` named `product.jsx`.

This component is going to accept the data of an individual product as props and return a row.

**product.jsx**
```
import React from "react";

const Product = (props) => {
    const {name, image, price, quantity} = props;
    
    return (
        <div className="bulk-form-row">
        <div className="bulk-form-col">
            <img src={image} className="bulk-product-image"/>
        </div>
        <div className="bulk-form-col">{name}</div>
        <div className="bulk-form-col">{price}</div>
        <div className="bulk-form-col">{quantity}</div>
        </div>
    )
}

export default Product;
```

Returning to `product-group.jsx`, we can map over the props and pass them into multiple `Product` components. First, we need to import the `Product` component, then write a function that returns the result of mapping over the product data. We'll call the result `productRows`, then include that after the first div with the class `bulk-form-row`. Each `Product` component returns another `bulk-form-row` that will appear below the headers.

**product-group.jsx**
```
import React from "react";
import Product from "./product";

const ProductGroup = (props) => {
    const productRows = props.products.map((product, index) => {
        return (
            <Product 
              key={index}
              name={product.name}
              image={product.image}
              price={product.price}
              quantity={product.quantity}
            />
        )
    });

    return (
        <div className="bulk-form-field">
            <div className="bulk-form-row">
                <div className="bulk-form-col"></div>
                <div className="bulk-form-col"><strong>Product</strong></div>
                <div className="bulk-form-col"><strong>Price</strong></div>
                <div className="bulk-form-col"><strong>Quantity</strong></div>
            </div>
            { productRows }
            <div className="bulk-form-row">
                <div className="bulk-form-col message"></div>
                <button className="button button--primary bulk-add-cart">Add to Cart</button>
            </div>
        </div>
    )
}

export default ProductGroup;
```


[screenshot showing images, product names, prices, and quantity]

That looks a bit better! We have product images, names, prices, and a quantity value populating on the page. Now we need to add functionality to adjust quantities and ultimately add the items to the cart.

In `bulk-order-form.jsx` we'll write a function that will accept a quantity and product id, allowing the user to update the quantity in our component state for a particular product. Within the constructor, add the following code:
```
this.updateQuantity = (quantity, id) => {
    const products = [...this.state.products];
    quantity = parseInt(quantity);
    if (quantity >= 0) {
        products.forEach(product => {
            product.id === id ? product.quantity = quantity : null
        });
        this.setState({
            products: products
        })
    }
}
```

First, we copy the array of product objects from the state so that we can update the nested quantity values in our copy, then apply the copied array to the state. This is necessary because nested values in state can't be updated directly.

The quantity is coerced into an integer, since we will be using a text field for our quantity boxes, but we will need to send the quantity as an integer in the payload to create a cart using the storefront API. There’s also a condition to make sure the quantity can't be decremented below 0, since we don't want users to be able to set a negative quantity value.

We need to pass the `updateQuantity` function down to our individual product rows. Let’s pass that as props to `ProductGroup`:

**bulk-order-form.jsx**
```
render() {
    return (
        <div>
            <ProductGroup 
                products={this.state.products}
                updateQuantity={this.updateQuantity}
                />
        </div>
    )
}
```

Within `product-group.jsx`, we can then pass the `updateQuantity` function to our `Product` components. We need to pass the product id too so we can tie quantity changes to a specific product.


**product-group.jsx**
```
const productRows = props.products.map((product, index) => {
    return (
        <Product 
            key={index}
            name={product.name}
            product_id={product.id}
            image={product.image}
            price={product.price}
            quantity={product.quantity}
            updateQuantity={props.updateQuantity}
        />
    )
});
```

In the `Product` component, we need to pass the `updateQuantity` function and `product_id` props. We’re also going to add up and down arrows, along with a text input field so a shopper can enter the quantity manually or increment the quantity 1 at a time. The button classes and the arrow icons are borrowed from the default Cornerstone theme, so they look like the quantity selector you’d normally see on any individual product page.

**product.jsx**
```
import React from "react";

const Product = (props) => {
    const {name, product_id, image, price, quantity, updateQuantity} = props;
    return (
        <div className="bulk-form-row">
            <div className="bulk-form-col">
                <img src={image} className="bulk-product-image"/>
            </div>
            <div className="bulk-form-col">{name}</div>
            <div className="bulk-form-col">{price}</div>
            <div className="bulk-form-col">
                <div className="form-increment">
                <button className="button button--icon" onClick={() => updateQuantity(quantity - 1, product_id)}>
                    <span className="is-srOnly">Decrease Quantity:</span>
                    <i className="icon" aria-hidden="true">
                    <svg viewBox="0 0 24 24" id="icon-keyboard-arrow-down" width="100%" height="100%"><path d="M7.41 7.84L12 12.42l4.59-4.58L18 9.25l-6 6-6-6z"></path></svg>
                    </i>
                </button>
                <input type="text" className="form-input form-input--incrementTotal" value={quantity} onChange={(e) => {updateQuantity(e.target.value, product_id)}}></input>
                <button className="button button--icon" onClick={() => updateQuantity(quantity + 1, product_id)}>
                    <span className="is-srOnly">Increase Quantity:</span>
                    <i className="icon" aria-hidden="true">
                    <svg viewBox="0 0 24 24" id="icon-keyboard-arrow-up" width="100%" height="100%"><path d="M7.41 15.41L12 10.83l4.59 4.58L18 14l-6-6-6 6z"></path></svg>
                    </i>
                </button>
                </div>
            </div>
        </div>
    )
}

export default Product;
```
Notice that we are passing the `updateQuantity` function to the onClick handler of both the increment and decrement buttons. This function needs to take a quantity and a product ID, so we pass the current quantity plus or minus 1 as well as the product ID prop of the `Product` component.

The quantity now updates when you click the up or down arrows! If you're still using `console.log(this.state)` in the render method in `bulk-order-form.jsx`, you should see the product quantities update in the state.`

We've got all our products appearing and the quantities are adjustable. Now we need to add real functionality to the add to cart button. This is where the storefront cart API comes in.

Let's go back to `bulk-order-form.jsx`. Add a new method within the constructor called `addToCart`. The cart API payload needs an array of product objects including the product id and quantity. Since the property is called `lineItems`, we'll name our variable the same thing. In the code below, the products stored in state are mapped to a new array if they have at least 1 quantity. Mapping an array this way will return the same number of values as the original array, but with null values present at all indices where the condition is false. To clean this up, we also apply a filter to return an array free of null values.

**bulk-order-form.jsx**
```
this.addToCart = () => {
    const lineItems = this.state.products.map(product => {
        if (product.quantity > 0) {
            return {
                productId: product.id,
                quantity: product.quantity
            }
        }
    }).filter(item => item != null);
}
```

We need to check if any of the `lineItems` products have a quantity before doing anything. We can set up a condition based on the length of the `lineItems` array, then make a request to the cart API to check if a cart already exists, or if we need to create a new one. 

**bulk-order-form.jsx**

```
this.addToCart = () => {
    const lineItems = this.state.products.map(product => {
        if (product.quantity > 0) {
            return {
                productId: product.id,
                quantity: product.quantity
            }
        }
    }).filter(item => item != null);

    if (lineItems.length > 0) {
        fetch(`/api/storefront/cart`)
        .then(response => response.json())
        .then(cart => console.log(cart))
    }
}
```
If there are any values in `lineItems`, we make the request to retrieve the existing cart on the storefront. When a cart doesn't exist, this API request returns an empty array. Otherwise, we can extract the unique cart ID from the response.

We can assign this function to an onClick handler on the add to cart button to test how the results from the cart API will appear. First, pass the `addToCart` function as a prop in `ProductGroup`:

**bulk-order-form.jsx**

```
render() {
        return (
            <div>
                <ProductGroup 
                    products={this.state.products}
                    updateQuantity={this.updateQuantity}
                    addToCart={this.addToCart}
                    />
            </div>
        )
    }
}
```

In `product-group.jsx` we can then use the addToCart function passsed as a prop and attach it to the add to cart button's onClick handler:

**product-group.jsx**
```
return (
        <div className="bulk-form-field">
            <div className="bulk-form-row">
                <div className="bulk-form-col"></div>
                <div className="bulk-form-col"><strong>Product</strong></div>
                <div className="bulk-form-col"><strong>Price</strong></div>
                <div className="bulk-form-col"><strong>Quantity</strong></div>
            </div>
            { productRows }
            <div className="bulk-form-row">
                <div className="bulk-form-col message"></div>
                <button className="button button--primary bulk-add-cart" onClick={props.addToCart}>Add to Cart</button>
            </div>
        </div>
    )
```

When you click the add to cart button, you should now see the results of the cart API request in your browser console. If there's no cart currently, you should see an empty array. Otherwise you can expand the array returned to see the cart details, including the cart ID.

[screenshots showing what appears if a cart does and doesn't exist]


We'll write two separate functions to deal with the results, creating a new cart with the user selection or adding them to an existing cart. Instead of simply logging the result of the cart API request, we'll take the results and determine whether to invoke either function. We'll also log any errors if there are issues adding items to the cart.

**bulk-order-form.jsx**

```
this.addToCart = () => {
            const lineItems = this.state.products.map(product => {
                if (product.quantity > 0) {
                    return {
                        productId: product.id,
                        quantity: product.quantity
                    }
                }
            }).filter(item => item != null);

            if (lineItems.length > 0) {
                this.setState({message: "Adding items to your cart..."});

                fetch(`/api/storefront/cart`)
                .then(response => response.json())
                .then(cart => {
                    if(cart.length > 0) {
                        return addToExistingCart(cart[0].id)
                    } else {
                        return createNewCart()
                    }
                })
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
    }
```

If no cart is returned, we can make a POST to the `/api/storefront/carts` endpoint with our lineItems to create a new cart with the user’s selections. Otherwise, we pass the cart ID as a parameter to `/api/storefront/carts/{cart_id}/items` with the same payload. Should either of the requests fail, we log an error to the browser console. Otherwise, we log the response from the cart API, which should include all the current cart details.

By now, you should be able to add multiple items to the cart at the same time. However, the user experience is a bit lacking - there's no visible letting you know items have been added. We should tell shoppers items are being added to the cart, then redirect them to the cart page once the API request has finished.

There is an empty column on the add to cart button row where we can display messages to shoppers. Let's add a message property to the state so we can update the messages when necessary. In the addToCart method in `bulk-order-form.jsx` we can set a message indicating items are being added to the cart. If no quantity is specified for any product, we should advise the shopper to set a quantity.

We can also handle the redirect by adding another `.then` handler in the promise chain after the API requests have finished.

**bulk-order-form.jsx**
```
if (lineItems.length > 0) {
        this.setState({message: "Adding items to your cart..."});

        fetch(`/api/storefront/cart`)
        .then(response => response.json())
        .then(cart => {
            if(cart.length > 0) {
                return addToExistingCart(cart[0].id)
            } else {
                return createNewCart()
            }
        })
        .then(() => window.location = "/cart.php")
        .catch(err => console.log(err))
    } else {
        this.setState({message: "Please select a quantity for at least 1 item"});
    }
```

Since we're now setting a message in state, we can pass this message as props to `ProductGroup` and display it in the column to the left of the add to cart button.

**bulk-order-form.jsx**

```
render() {
    return (
        <div>
            <ProductGroup 
                products={this.state.products}
                updateQuantity={this.updateQuantity}
                addToCart={this.addToCart}
                message={this.state.message}/>
        </div>
        )
    }
}
```

**product-group-jsx**
```
return (
        <div className="bulk-form-field">
            <div className="bulk-form-row">
                <div className="bulk-form-col"></div>
                <div className="bulk-form-col"><strong>Product</strong></div>
                <div className="bulk-form-col"><strong>Price</strong></div>
                <div className="bulk-form-col"><strong>Quantity</strong></div>
            </div>
            { productRows }
            <div className="bulk-form-row">
                <div className="bulk-form-col message"><strong>{props.message}</strong></div>
                <button className="button button--primary bulk-add-cart" onClick={props.addToCart}>Add to Cart</button>
            </div>
        </div>
    )
```

We can go one step further and disable the add to cart button to avoid multiple clicks on the add to cart button. In the addToCart method, we can select the button and adjust whether it is disabled afterwards.

**bulk-order-form.jsx**

```
this.addToCart = (e) => {
    const button = e.target;
```

```
 if (lineItems.length > 0) {
    button.disabled = true;

```

If add to cart fails for any reason, we can re-enable the button too. Instead of logging the error message to the console, we can pass it as a message.

**bulk-order-form.jsx**

```
fetch(`/api/storefront/cart`)
    .then(cart => cart.json())
    .then(data => {
        if(data.length > 0) {
            return addToExistingCart(data[0].id)
        } else {
            return createNewCart()
        }
    })
    .then(() => window.location = "/cart.php")
    .catch(err => handleFailedAddToCart(err, this, button))
```

Instead of console.log, we're passing the error message to a function called `handleFailedAddToCart`, which lives within the scope of the `addToCart` method.


```
function handleFailedAddToCart(message, self,  button) {
        self.setState({
            message: message
        });
        return button.disabled = false;
    }
```

We can also clear any error messages if a shopper decides to modify one of the item quantities. In the updateQuantity method we can reset the message.

**bulk-order-form.jsx**

```
this.updateQuantity = (quantity, id) => {
            const products = [...this.state.products];
            quantity = parseInt(quantity);
            if (quantity >= 0) {
                products.forEach(product => {
                    product.id === id ? product.quantity = quantity : null
                });
                this.setState({
                    products: products,
                    message: ''
                })
            }
        }
```

[gif showing different messages appearing]

Now we have a working form to enable shoppers to add items to their cart in bulk! In part 2, I'll cover how to create a bulk order form for product pages to add multiple variants to the cart at the same time.