# Flex

## About

This tutorial walks through all aspects of the 'Flex' codebase. We start by comparing and contrasting general architecture against our legacy code base. Afterwords, we deep dive into the core functionality of Flex, explore modular design patterns, ...

## 1.0: Termonology and architecture:

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
    * A shop on each cluster which acts as a central responsitory for velocity templates Globally scoped, templates and methods accessible to all shops

## 2.0 Flex code base:

### About
* Coined *Flex* after the CSS layout, *Flexbox*. For Clients, Sales Reps and Program managers, this distinguishes the fact that our sites are responsive and mobile friendly. Bottom line: provide the best possible customer experience that's as simple and smooth as possible, end-to-end. 
* Development began a couple of years back. As of 2020, all new sites are developed on C3.
* Flex employs modular design patterns (components, dependencies and partials)
* Syntatically Awesome Stylesheets (SASS) is used on all Flex sites.

### Architecture:

There are three fundemental concepts responsible for supporting modularity on Flex by providing a layer of abstraction on top of static velocity templates:

#### partials

lib_macros_partials

#### dependencies

Lib_macros_dependencies.vm


#### components

lib_macros_components
				
