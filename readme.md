# NodeJS API with GraphQL
- Rest stands for Representational State Transfer
- API end points: GET, POST, PUT, PATCH, DELETE
- RESTful API is stateless. No state is stored for requests. All requests are treated as standalone.
- This project shows how to create NodeJS API backend with expressJS. It also contains 
  - basic CRUD in NodeJS
  - upload images to local storage
  - authentication of user
  - validation of forms
  - error handling
- Workflow: App.js -> routes -> controllers -> third party libraries and/or external services like mongoDB -> respond with success or failure -> return status code and data
- recall "app.use" again


## Getting Started
- `npm init`, `git init`, `npm install --save express`, `npm install --save-dev nodemon`, `npm install --save body-parser`, `npm install --save multer`, `npm install --save bcryptjs`, `npm install --save jsonwebtoken`, `npm install --save socket.io`
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
- to use postman, `npm start` script then go to postman and put the url with te endpoints method (i.e. GET, POST etc).


## Scaffolding to accept basic API requests
- then create "routes/feed.js" and "controllers/feed.js"
- validate requests in routes
```
// routes/feed.js with validation for requests too
router.post(
  '/post',
  isAuth,
  [
    body('title')
      .trim()
      .isLength({ min: 5 }),
    body('content')
      .trim()
      .isLength({ min: 5 })
  ],
  feedController.createPost
);

/controller/feed.js to create post
exports.createPost = (req, res, next) => {
  // validation with express validation requests
  // raise/throw error if validation not met
  // set the data
  // create post with data
  // save post in database with mongoose
  // return res.status if success.
  // catch all errors
  // send errors to middleware in app.js
};
```


## Validation Requests
- Validating results and sending back errors will need the use of express's validation requests
```
// /controllers/feed.js
const { validationResult } = require('express-validator/check');

exports.signup = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    const error = new Error('Validation failed.');
    error.statusCode = 422;
    error.data = errors.array();
    throw error;
  }
  ...

// app.js
app.use((error, req, res, next) => {
  console.log(error);
  const status = error.statusCode || 500;
  const message = error.message;
  const data = error.data;
  res.status(status).json({ message: message, data: data });
});  
```


## Create Object
- create an object in `post` model folder with mongoose library as schema constructor
- then in controller, create a data entry with mongoose `post.save();` referring the `post` object created from model


## Error Handling
- in controller if request fails, throw an error with the `error.statusCode` explicitly stated. Then have an error handling function aka error handling middleware to handle the error
```
// /controller/feed.js
...
if (!errors.isEmpty()) {
    const error = new Error('Validation failed, entered data is incorrect.');
    error.statusCode = 422;
    throw error;
}
...
.catch(err => {
    if (!err.statusCode) {
    err.statusCode = 500;
    }
    next(err);
});

// app.js
app.use((error, req, res, next) => {
  console.log(error);
  const status = error.statusCode || 500;
  const message = error.message;
  const data = error.data;
  res.status(status).json({ message: message, data: data });
});
```


## Handling image uploads
- Handling Image uploads by storing files in local drive, then accessing it from controller.
```
// app.js
const multer = require('multer');
...
const fileStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'images');
  },
  filename: (req, file, cb) => {
    cb(null, new Date().toISOString() + '-' + file.originalname);
  }
});
...
const fileFilter = (req, file, cb) => {
  if (
    file.mimetype === 'image/png' ||
    file.mimetype === 'image/jpg' ||
    file.mimetype === 'image/jpeg'
  ) {
    cb(null, true);
  } else {
    cb(null, false);
  }
};
...
app.use(
  multer({ storage: fileStorage, fileFilter: fileFilter }).single('image')
);

// then access the file from local storage
```


## Authentication
- client send auth data to server -> once authenticated, a token can be sent to client storage (i.e. browser cookies) -> the same token is stored in backend server storage to keep track of sessions.
- to start user authentication, install `npm install --save bcryptjs`
- then require bcrypt during user sign up function method -> then use bcrypt to hash password -> save the hashed password

### JWT
- JSON Web Token (JWT) is JSON Data + Signature. The signature can only be verified by the server.
- can go to "https://jwt.io" to check what is the content of the JSON data and check if the JWT token hash value is correct/matches, however, the "super long secret private key" is unknown. Therefore, the generated JWT token hash would be different without the proper "super long secret private key".
- 1. Steps on how to create JWT for authentication
  - User logs in with correct username and password
  - in the server, details like email and password is matched
  - then a JWT token is generated with a secret key along with expiry
  - the JWT token is then sent to the user's client browser to be stored for future validation
- 2. Steps on how to validate JWT token
  - client front end making API request with JWT value as Authorization in header
  - middleware in backend will verify the requests by checking/decoding the Authorization value in header, catch any error if fails, else `next();`
  - in routes, import the result of the validation function from the middleware and put into the routes
  - once JWT verified, API request and continue
- Example implementation
```
// /controllers/auth.js to create jwt
exports.login = (req, res, next) => {
    // get req body details
    // find the user details like email and password
    // validate if user details are correct, else raise error
    // use bcrypt to compare password given and user password, raise error if mistake
    // generate a JWT with `const token =jwt.sign({ ... some json data ... }, "some super long secret private key", {expiresIn: "1h"});`
    // return token with res.status(200).json({token: token, userId: id});
};

// /middleware/is-auth.js to validate jwt
const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  const authHeader = req.get('Authorization'); // get Authorization from REST API's header
  if (!authHeader) {
    const error = new Error('Not authenticated.');
    error.statusCode = 401;
    throw error;
  }
  const token = authHeader.split(' ')[1];
  let decodedToken;
  try {
    decodedToken = jwt.verify(token, 'somesupersecretsecret');
  } catch (err) {
    err.statusCode = 500;
    throw err;
  }
  if (!decodedToken) {
    const error = new Error('Not authenticated.');
    error.statusCode = 401;
    throw error;
  }
  req.userId = decodedToken.userId;
  next();
};

// /routes/feed.js
const isAuth = require('../middleware/is-auth');

router.get('/posts', isAuth, feedController.getPosts);
```


## Creating relations between two documents
- in the schema of a model, add a field with type `Schema.Types.ObjectId` to add id value and it also references the model name of the id
- Example implementation below:
```
// /models/post.js
creator: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
}
```


## Async/Await
- normally javascript doesn't wait for a statement to complete. It will only wait if your statement is in `.then().catch` or in `async/await`
- because `async/await` doesn't have catch, you will have to wrap `await` in `try {} catch (error) {}`
- version 14.3 NodeJS `await` promise can be outside of `async`. Previously `await` has to be in `async`
- example implementation
```
exports.getPosts = async (req, res, next) => {
  const currentPage = req.query.page || 1;
  const perPage = 2;
  let totalItems;
  try {
    const totalItems = await Post.find().countDocuments();
    const posts = await Post.find()
      .skip((currentPage - 1) * perPage)
      .limit(perPage);

    res.status(200).json({
      message: 'Fetched posts successfully.',
      posts: posts,
      totalItems: totalItems
    });
  } catch (error) {
    if (!err.statusCode) {
      err.statusCode = 500;
    }
    next(err);
  }
};
```


## Websocket and socket.io
- Sometimes it is better to use websocket instead of http. http is a method in which you send a request (req) and get a response (res)
- websockets it is data being pushed from the server to the client without request (req) from the client
- `npm install --save socket.io` socket.io needs to be installed in the backend (nodeJS) and the frontend (reactJS)
- after connection with mongodb database is setup, and after the server is launched, then you establish a socket.io connection. Also in the client front end, install `npm install --save socket.io-client`
- below is example implementation of showing all posts to the app user.
```
// socket.js in NodeJS
let io;

module.exports = {
  init: httpServer => {
    io = require('socket.io')(httpServer);
    return io;
  },
  getIO: () => {
    if (!io) {
      throw new Error('Socket.io not initialized!');
    }
    return io;
  }
};

// app.js in NodeJS
mongoose
  .connect(
    'mongodb+srv://maximilian:9u4biljMQc4jjqbe@cluster0-ntrwp.mongodb.net/messages?retryWrites=true'
  )
  .then(result => {
    const server = app.listen(8080);
    const io = require('./socket').init(server);
    io.on('connection', socket => {
      console.log('Client connected');
    });
  })
  .catch(err => console.log(err));

// src/pages/feed/feed.js in ReactJS
import openSocket from 'socket.io-client'

const socket = openSocket('http://localhost:8080');
socket.on('posts', data => {
    if (data.action === 'create') {
    this.addPost(data.post);
    } else if (data.action === 'update') {
    this.updatePost(data.post);
    } else if (data.action === 'delete') {
    this.loadPosts();
    }
});
```
- import `socket.js` files into all controllers that you want to use websocket
```
// /controllers/feed.sj
const io = require("../socket");

// in the relevant methods like exports.createPost
io.getIO().emit('posts', {
    action: 'create',
    post: { ...post._doc, creator: { _id: req.userId, name: user.name } }
});

// or exports.updatePost
io.getIO().emit('posts', { action: 'update', post: result });

// /src/pages/feed/feed.js 
// listen in to any websocket push from NodeJS. ensure that the same event name between front end and back end. In this case it is "posts". thereafter any data after the event name is determined by the coder.
socket.on('posts', data => {
    if (data.action === 'create') {
    this.addPost(data.post);
    } else if (data.action === 'update') {
    this.updatePost(data.post);
    } else if (data.action === 'delete') {
    this.loadPosts();
    }
});

// then in the relevant methods in ReactJS, execute the code to add, update or delete the post.

```
- alternative websocket for express [https://www.npmjs.com/package/express-ws](https://www.npmjs.com/package/express-ws)
