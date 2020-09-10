---
layout: post
title:  "A REST api using mongodb and node"
author: VJ
categories: [ Microservices ]
tags: [mongodb, node, express, javascript, CRUD, REST, API]
image: assets/images/node-mongo.jpeg
description: "A REST api using mongodb and nodejs"
featured: true
hidden: true
---

When it comes to quickly whipping up a REST api for a POC, I love the combination of express, mongodb and nodejs. It is lightweight and easy to develop with a quick turn-around time for a POC devlopment.

I have a demo project for nodejs REST api, which uses express as the server and mongodb as datastore. it is using mongoose to do the crud operations.

### package.json

```
    "express": "^4.17.1",
    "mongoose": "^5.9.27",
```

### Express server 

Is listening on port 3000. You can change this by editing the following line in app.js

```javascript
app.listen(3000, () => {
    console.log('Express started at port 3000');
})
```

Express router helps in routing the requests to various resources in the controller

```javascript
var productRouter = new express.Router();
```

### productController.js

- productRouter.get('/products' : gets the list of all products.
- productRouter.get('/products/:name' : gets the details of a particular product.
- productRouter.post('/products' : create a new product.
- productRouter.delete('/products/:name' : delete a particular product.
- productRouter.put('/products/:name' : updates an existing product.

### mongoose.js

Contains the configuration for mongoose to connect to mongodb. It is currently hardcoded to my localhost mongodb. I prefer to run a docker container on my machine rather than installing everything.


Mongoose schema is defined in the product.js file. To read about mongoose visit [mongoose](https://mongoosejs.com/docs/guide.html)

This easy it is to develop a CRUD REST api. I think this is one of the coolest way to whip up a quick API.

The project also contains a json file for the postman tests.

You can find the project at [gitlab](https://gitlab.com/gunnervj/product-rs)
