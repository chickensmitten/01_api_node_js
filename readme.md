# NodeJS API with ExpressJS
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


## Deployment
### Preparing for deployment
- deployment checklist
  - using environment variables
  - production API keys
  - reduce error output details
  - set secure reponse headers
  - add asset compression (sometimes handled by hosting provider)
  - configure logging (sometimes handled by hosting provider)
  - use SSL/TLS (sometimes handled by hosting provider)

#### Environment variables
- change MongoDB URI database
```
// /app.js
const MONGODB_URI = `mongodb+srv://${process.env.MONGO_USER}:${
  process.env.MONGO_PASSWORD
}@cluster0-ntrwp.mongodb.net/${process.env.MONGO_DEFAULT_DATABASE}`;
```
- change `app.listen(3000)` to `app.listen(process.env.PORT || 3000);`
- change `const stripe = require('stripe')(process.env.STRIPE_KEY);`
- add the environment variables in `nodemon.json`
```
// /nodemon.json
{
    "env": {
        "MONGO_USER": "maximilian",
        "MONGO_PASSWORD": "9u4biljMQc4jjqbe",
        "MONGO_DEFAULT_DATABASE": "shop",
        "STRIPE_KEY": "sk_test_T8OE02SHDZWLwk4TYtrWlsat"
    }
}
```
- add a start script `"start:dev": "nodemon app.js"` for dev environment
- for production start script use `"start": "NODE_ENV=production MONGO_USER=maximilian MONGO_PASSWORD=9u4biljMQc4jjqbe MONGO_DEFAULT_DATABASE=shop STRIPE_KEY=sk_test_T8OE02SHDZWLwk4TYtrWlsat node app.js",`
- in ".gitignore" ignore the `nodemon.json` so that the info won't be pushed to github

#### setting secure headers
- `npm install --save helmet` it will secure all incoming requests with special headers. Can see this in browser's network
- then in app.js add `const helmet = require('helmet'); app.use(helmet())`

#### compressing assets
- `npm install --save compression`
- `const compression = require('compression'); app.use(compression());`

#### configure logging
- `npm install --save morgan`, this will show logging data in console.
- Example code implementation will log into a file called "access.log"
```
const morgan = require('morgan'); 
const accessLogStream = fs.createWriteStream(
  path.join(__dirname, 'access.log'),
  { flags: 'a' }
);
app.use(morgan('combined', { stream: accessLogStream }));
```

#### SSL/TLS
- in terminal call `openssl req -nodes -new -x509 -keyout server.key -out server.cert`. once enter, fill up the question. if using in development, have to set "common name" to "localhost".
- once done, "server.cert" and "server.key" files will be generated. then add https with 
```
const https = require('https'); 

...

const privateKey = fs.readFileSync('server.key'); 
const certificate = fs.readFileSync('server.cert');

...

https
  .createServer({ key: privateKey, cert: certificate }, app)
  .listen(process.env.PORT || 3000);
```
-

### Deployment to Heroku
- use Heroku CLI, refer to Heroku CLI instructions in Heroku website
- `heroku login`
- `heroku git:remote -a <project name>`
- setup script in package.json
```
  "engines": {
    "node": "10.9.0"
  },
```
- in Procfile add `web: node app.js` to only execute the app.js file
- remember to ignore files like .cert, .key, node modules, env variables etc.
- `git push heroku master`
- Go to "Config Vars" in Heroku then add all the production values in Heroku
- in MongoDB Atlas, need to whitelist IP from Heroku. Refer to this URL for Heroku's [whitelist IP](https://help.heroku.com/JS13Y78I/i-need-to-add-heroku-dynos-to-our-allowlist-what-are-ip-address-ranges-in-use-at-heroku)
- A note on storing files in Heroku
```
Storing User-generated Files on Heroku
Here's one important note about hosting our app on Heroku!

The user-generated/ uploaded images, are saved and served as intended. But like all hosting providers that offer virtual servers, your file storage is not persistent!

Your source code is saved and re-deployed when you shut down the server (or when it goes to sleep, as it does automatically after some time in the Heroku free tier).

But your generated and uploaded files are not stored and re-created. They would be lost after a server restart!

Therefore, it's recommended that you use a different storage place when using such a hosting provider.

In cases where you run your own server, which you fully own/ manage, that does of course not apply.

What would be alternatives?

A popular and very efficient + affordable alternative is AWS S3 (Simple Storage Service): https://aws.amazon.com/s3/

You can easily configure multer to store your files there with the help of another package: https://www.npmjs.com/package/multer-s3

To also serve your files, you can use packages like s3-proxy: https://www.npmjs.com/package/s3-proxy

For deleting the files (or interacting with them on your own in general), you'd use the AWS SDK: https://aws.amazon.com/sdk-for-node-js/
```


### Additional deployment instructions
```
Useful Resources & Links
Attached, you find the source code for this section.

When using my source code, make sure to run npm install in the extracted folder!

Useful resources:

Herokus Docs: https://devcenter.heroku.com/categories/reference

Deploying SPAs (like our React App): https://medium.com/@baphemot/understanding-react-deployment-5a717d4378fd

Alternative Hosting Providers:

Amazon Web Services: https://aws.amazon.com/getting-started/projects/deploy-nodejs-web-app/

DigitalOcean: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04

And of course everything Google yields on "nodejs hosting"
```