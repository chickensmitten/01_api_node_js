# NodeJS API
- Rest stands for Representational State Transfer
- API end points: GET, POST, PUT, PATCH, DELETE
- ATTENTION: Gotcha, when running `npm start` in VSCode terminal, debugger runs. While it doesn't in a terminal outside of VSCode

## Getting Started
- `npm init`, `git init`, `npm install --save express`, `npm install --save-dev nodemon`, `npm install --save body-parser`
- in package.json file, change ["scripts"]["start"] with nodemon `"start": "nodemon app.js"`
- make changes to app.js to initialize the api routes
```
const express = require('express');
const bodyParser = require('body-parser');
const feedRoutes = require('./routes/feed');
const app = express();
// app.use(bodyParser.urlencoded()); // x-www-form-urlencoded <form>
app.use(bodyParser.json()); // application/json

app.use((req, res, next) => {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'OPTIONS, GET, POST, PUT, PATCH, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    next();
});

app.use('/feed', feedRoutes);
app.listen(8080);
```
- then create "routes/feed.js" and "controllers/feed.js"
- to use postman, `npm start` script then go to postman and put the url with te endpoints method (i.e. GET, POST etc).
