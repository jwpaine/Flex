# Flex

## About

This tutorial walks through all aspects of the 'Flex' codebase. We start by comparing and contrasting general architecture against our legacy code base. Afterwords, we deep dive into the core functionality of Flex, explore modular design patterns, ...

## Termonology and General Architecture:

* **C8**
    Avetti's Commerce-8 platform: a multi-vendor ecommerce platform.
* **cluster**: 
    * a cluster denotes a logical grouping of EC2 instances (virtual servers) comprising staging (preview) and production (shop) environments. C3 denotes *cluster 3* and the staging/preview environment is ggc8admin3.avetti.ca and the production/shop environment is ggc8prod3.avetti.ca
    * Backend-driven: 
        * API built on Java/Spring. 
        * Velocity (Java-based templating engine) renders VTL templates into markup server-side, which is returned as an HTTP response
        * Templates and shop-specific data stored in MySQL database. We interface with templates via Avetti's built-in template editor, or through a Sublime plugin which abstracts away the underlying template data stored in MySQL.
        * NGINX is used as a reverse-proxy, allowing clients to interface with backend API, and for serving static content (images, stylesheets), and handles domain mapping.
* **shop**
    * A discrete site containing it's own set of velocity templates, configuration, and static files
    * Each shop has a corrosponding Vendor ID (vid)
    * On C8, a shop's static files are located in the directory: /avetti/httpd/htdocs/content/preview/store/{vid}
    * css files are stored in ../{vid}/assets/themes/blaze_en/css"
* **Master Template Library (MTL)**
    * A shop on each cluster which acts as a central responsitory for velocity templates Globally scoped, templates and methods leveraged by all shops on a cluster.

## Flex MTL:

### About
* Coined *Flex* after the CSS layout, *Flexbox* to distinguish the fact that our sites are responsive and mobile friendly. 
* As of 2020, all new sites are developed in Flex.
* Flex employs modular design patterns (components, dependencies and partials)
* Syntatically Awesome Stylesheets (SASS) is used on all Flex sites.

### Architecture

The Flex MTL codebase is seperated into two parts: **velocity templates**, and **skeleton library**.
On the velocity template side, there are three core patterns on Flex, each of which provide a layer of abstraction over underlying static templates:

---
1. #### Partials

Velocity provides the macro `#parse`, which allows us to parse in a named template, directly. 

* Use `#parse("template_name.vm")` when parsing templates stored in the MTL

* Use `#parse("/$vendorSettingsDTO.vendorId/$vendorSettingsDTO.themeId/template_name.vm")` when parsing templates, local to your shop.

Building on top of the `#parse()` macro, **partials** are used to help de-couple template parsing. Take for example the category.vm template for a shop on C1 or C2: this template parses in a template from the MTL which is responsible for rendering category items, by calling `#parse("libpartCategoryProductList.vm")`. If you wish to modify functionality within libpartCategoryProductList.vm, without affecting other shops on the cluster, you you need to comment out that `#parse` statement, and replace it with the *contents of*  libpartCategoryProductList.vm -- 

For a Flex site, we're also relying upon an MTL template for render category items; however, we're calling `#renderPartials('category-content')` instead. When this `#renderPartials` macro is rendered, if local template `/$vendorSettingsDTO.vendorId/$vendorSettingsDTO.themeId/category-content.vm` was found when the partial was *initialized* when `#usePartials('category-content')` was called at the top of the template, than the content of that local template is rendered. If a local version was not found, than the global template `lib_category-content.vm` would be rendered. An error will offer when `#usePartials(partial_name)` renders, if neither an MTL nor local version exists. This general pattern allows us to create an alias for templates.

The MTL template `lib_macros_partials.vm` contains the code responsible for using and rendering partials. Open this template to explore the underlying mechanics if desired!

---
2. #### Dependencies

Making use of dependency patterns allows us to break coherent functionality off into it's own discrete ‘contract’, leading to code that is easier to manage, test, and re-use.

![dependency.png](dependency.png) 

#### Defining, Using, and Rendering Dependencies

**Define a dependency**:

```
#defineDependency(‘my-dependency’, {
	‘headMarkup’ : [
		“<script src=’...’></script>”,
“/$vendorSettingsDTO.vendorId/$vendorSettingsDTO.themeId/js_template.vm”,
] ,
	‘footMarkup’ : [
		“Asset_1”,
		“Asset_2”,
		     ...
		“Asset_n”
		] ,

‘requires’ : [‘jquery’, ‘dep_2’, … ‘dep_n’]

})
```
The `#defineDependency` macro is passed a dependency name, and a hashmap which may contain section-name/static-asset pairs along with a list of sub-dependencies required. Above, ‘headMarkup’ maps to a list containing a script and a local template. 

**Use and render**:
```
#useDependency(‘my-dependency’)
#renderDependencies(‘headMarkup’)
```
The `useDependency` macro flags a dependency as being used. Will render out any dependency sections, if `#renderDependencies(sectionName)` is called.

**Dependencies in action**:

![dependency_in_action.png](dependency_in_action.png) 

**Note**: when adding a new dependency to a local store, and not the MTL, include it as a new entry in `$configDependencies`, found at the bottom of template `configs.vm`

The MTL template `lib_macros_dependencies.vm` contains the code responsible for using and rendering dependencies. Open this template to explore the underlying mechanics if desired!

---
3. #### components

Components allow you to split the UI into independent, reusable pieces and provides abstraction over a predefined set of properties (DefaultSettings) and any underlying code describing how a segment of the user interface should be rendered. The key takeaway is modularity and reusability.

![components.png](components.png) 


**Creating a component**:

We create local component template `comp_componentName.vm` and call the `#registerComponent()` macro, which is defined in lib_macros_components.vm in the MTL. 

Component meta-data including author, version, and HelpURL are then set:

```
#registerComponent('componentName')
#setComponentAuthor('cpweb@geiger.com')
#setComponentVersion('1.0.0')
#setComponentHelpURL('')
```
We can use the `#addComponentDefaultSetting(‘name’, ‘value’)` macro to instantiate default settings. State is managed via component settings. Think of settings as private objects within the component:

```
#registerComponent('componentName')
#setComponentAuthor('cpweb@geiger.com')
#setComponentVersion('1.0.0')
#setComponentHelpURL('')
#addComponentDefaultSetting('settingName', ‘value’)
```
We define a render macro. componentName should match the registered component name and because macro names are globally scoped across the cluster, we are using vids to ensure names are unique. Within the render macro, we can use the `#getComponentSetting()` macro to return the value associated with a setting name.

```
#registerComponent('componentName')
#setComponentAuthor('cpweb@geiger.com')
#setComponentVersion('1.0.0')
#setComponentHelpURL('')
#addComponentDefaultSetting('settingName', ‘value’)

#macro(render_componentName_20181005604)
    #getComponentSetting('settingName')
#end
```

Testing macro is set:

```
#registerComponent('componentName')
#setComponentAuthor('cpweb@geiger.com')
#setComponentVersion('1.0.0')
#setComponentHelpURL('')
#addComponentDefaultSetting('settingName', ‘value’)

#macro(render_componentName_20181005604)

#end

#macro(test_componentName_20181005604)

#end

```

**Using a component**:

We can now use and render our component in another template on the site:

```
#useComponent('componentName')
```
Calling the #setComponentSetting macro allows us to override a componentDefaultSetting:

```
#useComponent('componentName')
#setComponentSetting('componentName', 'settingName', ‘value’)
```
`#renderedComponent` is called, which renders the component:

```
#useComponent('componentName')
#setComponentSetting('componentName', 'settingName', ‘value’)
#renderComponent('componentName')
```

The MTL template `lib_macros_components.vm` contains the code responsible for using and rendering dependencies. Open this template to explore the underlying mechanics if desired!

### skeleton library:

There are 23 javascript files which comprise the Flex Skeleton Library:

* ss.breakpoint-imaging.js
* ss.cache.js
* ss.categories.js
* ss.checkout-address.js
* ss.checkout-basket.js
* ss.checkout-payment.js
* ss.checkout-review.js
* ss.custom-modals.js
* ss.customers.js
* ss.custprops-admin.js
* ss.date.js
* ss.global.js
* ss.invoicing-admin.js
* ss.js
* ss.message-box.js
* ss.minicart.js
* ss.navBuilder.js
* ss.points-admin.js
* ss.price.js
* ss.product-detail.js
* ss.product-tabs.js
* ss.shoppergroup-admin.js
* ss.wishlist.js

Each of the files above are loaded through a seperate dependency, defined in `lib_configs.vm` in the Flex MTL. 
On the preview environment, these assets are located in the `/avetti/httpd/htdocs/content/preview/store/20170604234/assets/js` directory.

Each .js file is of the following form:

```
ss.dependency_name = (function (){

	var dependency_name = {}

    dependency_name.defaultConfigs = {
        'key_1' : 'value_1',
        ...,
        'key_n : 'value_n'
    }

    dependency_name.globalFunction = function() {

    }

    var localFunction = function() {

    }

    return dependency_name
})()
```

Along with the .js file on disk, the skeleton store dependencies defined in `lib_configs.vm` may also leverage a seperate velocity template used to initalize values in the .js file.

The `ss.product-detail` dependency is an example of this:

```
#defineDependency('ss.product-detail', {
	'requires': ['ss', 'ss.cache', 'ss.price', 'ss.minicart', 'dialog-polyfill', 'jquery', 'magic-zoom-plus', 'ss.message-box', 'spin'],
	'footMarkup': [
		"<script src='store/$configMTLVID/assets/js/ss.product-detail.js'></script>",
		"lib_js_product_detail.vm"
	]
})

```
On a shop's `item.vm` template, `#useDependency('ss.product-detail')` is called, the 9 other dependencies required by `ss.product-detail` are loaded, and when `#renderDependencies('footMarkup')` is rendered at bottom of the item.vm template, `store/$configMTLVID/assets/js/ss.product-detail.js` is rendered, alongside the content of MTL template `lib_js_product_detail.vm`.

Taking a look at the contents of lib_js_product_detail.vm reveals, amongst other things, the initalization of a number of default settings for product_detail:

```
	ss.product_detail.defaultConfigs.quantityVerbiage = "$esc.javascript("#showMessage('item.label.quantity')")";
	ss.product_detail.defaultConfigs.excludeProductImage = 'noimage.jpg';
    ss.product_detail.itemCode = "$esc.javascript($item.itemCode)";
	ss.product_detail.itemID = "$esc.javascript($item.itemid)";

    ...
```

## Your Flex Store:

I've setup a test site that you can modify and learn on:

When logging in via https://ggc8admin3.avetti.ca/preview/login.admin 

You'll find the website named `CJORDAN SANDBOX` with store id 20210422954

Clicking the store id will allow to you access settings for this Shop. From there, feel free to poke around. 'templates -> manage templates' will allow you to view all templates for this shop. Template `home.vm` is rendered when hitting your site link, on preview:

https://ggc8admin3.avetti.ca/preview/store.html?vid=20210422954


## Things to cover...

1. General store configuration

2. Defining new local dependencies

3. Changing default settings for components

5. Overriding an MTL dependency

6. Stylesheets: Skinning process, skinning tool

7. Transfering files via sFTP

8. Interfacing with velocity templates using sublime





