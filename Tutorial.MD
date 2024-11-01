# Cheatsheet for Job Management API

## `app.js`

### Line-by-Line Explanation

```javascript
require("dotenv").config();
```

- Loads environment variables from a `.env` file into `process.env`.

```javascript
require("express-async-errors");
```

- Allows handling asynchronous errors in Express without using `try/catch` blocks.

```javascript
const express = require("express");
const app = express();
```

- Imports the Express library and creates an instance of an Express application.

### Middleware and Routes

```javascript
// Error handlers
const notFoundMiddleware = require("./middleware/not-found");
const errorHandlerMiddleware = require("./middleware/error-handler");
```

- Imports custom middleware for handling 404 errors and general error responses.

```javascript
const connectDB = require("./db/connect");
const authenticateUser = require("./middleware/authentication");
```

- Imports the database connection function and authentication middleware.

```javascript
// Routes
const authRouter = require("./routes/auth");
const jobRouter = require("./routes/jobs");
```

- Imports routers for authentication and job management routes.

```javascript
app.use(express.json());
```

- Middleware to parse incoming JSON requests.

### Defining Routes

```javascript
app.use("/api/v1/auth", authRouter);
app.use("/api/v1/jobs", authenticateUser, jobRouter);
```

- Sets up the routes for authentication and job-related endpoints, protecting the job routes with authentication middleware.

### Error Handling Middleware

```javascript
app.use(notFoundMiddleware);
app.use(errorHandlerMiddleware);
```

- Registers the error handling middleware to catch and respond to errors.

### Server Initialization

```javascript
const port = process.env.PORT || 3000;
```

- Sets the server port from environment variables or defaults to 3000.

```javascript
const start = async () => {
  try {
    await connectDB(process.env.MONGO_URI);
    app.listen(port, () =>
      console.log(`Server is listening on port ${port}...`)
    );
  } catch (error) {
    console.log(error);
  }
};
```

- Asynchronously connects to the database and starts the Express server. Logs an error if the connection fails.

```javascript
start();
```

- Calls the `start` function to initiate the server.

---

## `controllers/job.js`

### Line-by-Line Explanation

```javascript
const Job = require("../models/Job");
const { StatusCodes } = require("http-status-codes");
const { BadRequestError, NotFoundError } = require("../errors");
```

- Imports the Job model and HTTP status codes. Imports custom error classes.

### Get All Jobs

```javascript
const getAllJobs = async (req, res) => {
  const jobs = await Job.find({ createdBy: req.user.userId }).sort("createdAt");
  res.status(StatusCodes.OK).json({ jobs, count: jobs.length });
};
```

- Retrieves all jobs created by the authenticated user, sorts them by creation date, and responds with the job list and count.

### Get a Specific Job

```javascript
const getJob = async (req, res) => {
  console.log(req.params);
  const {
    user: { userId },
    params: { id: jobId },
  } = req;
  const job = await Job.findOne({ _id: jobId, createdBy: userId });
  if (!job) {
    throw new NotFoundError(`No job with id ${jobId}`);
  }
  res.status(StatusCodes.OK).json({ job });
};
```

- Retrieves a specific job based on the job ID and the user ID. Throws an error if the job does not exist.

### Create a Job

```javascript
const createJob = async (req, res) => {
  req.body.createdBy = req.user.userId;
  const job = await Job.create(req.body);
  res.status(StatusCodes.CREATED).json({ job });
};
```

- Creates a new job using the request body and assigns it to the authenticated user. Responds with the created job.

### Update a Job

```javascript
const updateJob = async (req, res) => {
  const {
    body: { company, position },
    user: { userId },
    params: { id: jobId },
  } = req;
  if (company === "" || position === "") {
    throw new BadRequestError("Company or position field can not be empty");
  }
  const job = await Job.findByIdAndUpdate(
    { _id: jobId, createdBy: userId },
    req.body,
    { new: true, runValidators: true }
  );
  if (!job) {
    throw new NotFoundError(`No job with id ${jobId}`);
  }
  res.status(StatusCodes.OK).json({ job });
};
```

- Updates an existing job. Validates that the company and position fields are not empty. Responds with the updated job.

### Delete a Job

```javascript
const deleteJob = async (req, res) => {
  const {
    user: { userId },
    params: { id: jobId },
  } = req;
  const job = await Job.findByIdAndDelete({ _id: jobId, createdBy: userId });
  if (!job) {
    throw new NotFoundError(`No job with id ${jobId}`);
  }
  res.status(StatusCodes.OK).send();
};
```

- Deletes a job based on the job ID and the user ID. Responds with a success status if the job is deleted.

```javascript
module.exports = {
  getAllJobs,
  getJob,
  createJob,
  updateJob,
  deleteJob,
};
```

- Exports the job controller functions for use in routes.

---

## `controllers/auth.js`

### Line-by-Line Explanation

```javascript
const User = require("../models/User");
const { StatusCodes } = require("http-status-codes");
const { BadRequestError, UnauthenticatedError } = require("../errors");
```

- Imports the User model and HTTP status codes. Imports custom error classes.

### Register User

```javascript
const register = async (req, res) => {
  const user = await User.create({ ...req.body });
  const token = user.createJWT();
  res.status(StatusCodes.CREATED).json({ user: { name: user.name }, token });
};
```

- Registers a new user and generates a JWT token upon successful creation. Responds with the user name and token.

### Login User

```javascript
const login = async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password) {
    throw new BadRequestError("Please provide email and password");
  }
  const user = await User.findOne({ email });
  if (!user) {
    throw new UnauthenticatedError("Invalid credential");
  }
  const isPasswordCorrect = await user.comparePassword(password);
  if (!isPasswordCorrect) {
    throw new UnauthenticatedError("Invalid credential");
  }
  const token = user.createJWT();
  res.status(StatusCodes.OK).json({ user: { name: user.name }, token });
};
```

- Authenticates a user by verifying the email and password. Generates a JWT token upon successful login and responds with the user name and token.

```javascript
module.exports = {
  register,
  login,
};
```

- Exports the authentication controller functions for use in routes.

---

## `middleware/authentication.js`

### Line-by-Line Explanation

```javascript
const User = require("../models/User");
const jwt = require("jsonwebtoken");
const { UnauthenticatedError } = require("../errors");
```

- Imports the User model, the JWT library, and custom error classes.

### Authentication Middleware

```javascript
const auth = async (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer")) {
    throw new UnauthenticatedError("Authentication invalid");
  }
  const token = authHeader.split(" ")[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = { userId: payload.userId, name: payload.name };
    next();
  } catch (error) {
    throw new UnauthenticatedError("Authentication invalid");
  }
};
```

- Middleware that checks for a valid JWT token in the `Authorization` header. If valid, it adds the user information to the request object and calls `next()`; otherwise, it throws an authentication error.

```javascript
module.exports = auth;
```

- Exports the authentication middleware for use in routes.

---

## `middleware/error-handler.js`

### Line-by-Line Explanation

```javascript
const { CustomAPIError } = require("../errors");
const { StatusCodes } = require("http-status-codes");
```

- Imports the custom error class and HTTP status codes.

### Error Handler Middleware

```javascript
const errorHandlerMiddleware = (err, req, res, next) => {
  let customError = {
    statusCode: err.statusCode || StatusCodes.INTERNAL_SERVER_ERROR,
    msg: err.message || "something went wrong try again later",
  };
  if (err instanceof CustomAPIError) {
    return res.status(err.statusCode).json({ msg: err.message });
  }
  return res.status(customError.statusCode).json({ msg: customError.msg });
};
```

- Middleware that handles errors in the application, returning a JSON response with the appropriate status code and message.

```javascript
module.exports = errorHandlerMiddleware;
```

- Exports the error handling middleware for use in the application.

---

## `middleware/not-found.js`

### Line-by-Line Explanation

```javascript
const notFound = (req, res) => res.status(404).send("Route does not exist");
```

- Middleware that handles 404 errors by sending a "Route does not exist" message.

```javascript
module.exports = notFound;
```

- Exports the 404 error handling middleware for use in the application.

---

## `models/Job.js`

### Line-by-Line Explanation

```javascript
const mongoose = require("mongoose");
```

- Imports Mongoose for creating the Job model.

### Job Schema

```javascript
const JobSchema = new mongoose.Schema(
  {
    company: {
      type: String,
      required: [true, "Please provide company"],
    },
    position: {
      type: String,
      required: [true, "Please provide position"],
    },
    createdBy: {
      type: mongoose.Types.ObjectId,
      ref: "User",
      required: true,
    },
  },
  { timestamps: true }
);
```

- Defines the schema for the Job model with fields for company, position, and creator.

### Job Model

```javascript
module.exports = mongoose.model("Job", JobSchema);
```

- Exports the Job model for use in the application.

---

## `models/User.js`

### Line-by-Line Explanation

```javascript
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
```

- Imports Mongoose for creating the User model, Bcrypt for password hashing, and JWT for generating tokens.

### User Schema

```javascript
const UserSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, "Please provide name"],
    },
    email: {
      type: String,
      required: [true, "Please provide email"],
      unique: true,
    },
    password: {
      type: String,
      required: [true, "Please provide password"],
    },
  },
  { timestamps: true }
);
```

- Defines the schema for the User model with fields for name, email, and password.

### Password Hashing

```javascript
UserSchema.pre("save", async function () {
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});
```

- Middleware that hashes the user's password before saving to the database.

### JWT Generation

```javascript
UserSchema.methods.createJWT = function () {
  return jwt.sign(
    { userId: this._id, name: this.name },
    process.env.JWT_SECRET,
    { expiresIn: "30d" }
  );
};
```

- Method to generate a JWT for the user.

### Password Comparison

```javascript
UserSchema.methods.comparePassword = async function (candidatePassword) {
  const isMatch = await bcrypt.compare(candidatePassword, this.password);
  return isMatch;
};
```

- Method to compare a provided password with the hashed password in the database.

```javascript
module.exports = mongoose.model("User", UserSchema);
```

- Exports the User model for use in the application.

```

```

## Bcrypt Cheatsheet

### Overview

- **Purpose**: Securely hashes passwords before storing them in a database.
- **Security**: Resistant to brute-force attacks through adaptive hashing.

### Key Features

- **Adaptive Hashing**: Adjusts computational cost to increase security over time.
- **Salt Generation**: Creates a unique salt for each password to prevent rainbow table attacks.
- **Cross-Platform Compatibility**: Available in many programming languages.

### How It Works

1. **Salting**: Generate a random salt for each password.
2. **Hashing**: Combine the password with the salt and hash it.
3. **Storage**: Save the hashed password and salt in the database.
4. **Verification**: Hash the input password and compare with stored hash.

### Example Usage

#### Hashing a Password

```javascript
const bcrypt = require("bcryptjs");

const hashPassword = async (password) => {
  const salt = await bcrypt.genSalt(10); // Generate a salt
  const hashedPassword = await bcrypt.hash(password, salt); // Hash the password
  return hashedPassword;
};
```

#### Comparing a Password

```javascript
const comparePassword = async (candidatePassword, storedHash) => {
  const isMatch = await bcrypt.compare(candidatePassword, storedHash); // Compare passwords
  return isMatch;
};
```

---

## JWT (JSON Web Tokens)

### Overview

- **Purpose**: Securely transmits information between parties as a JSON object.
- **Common Use**: Authentication and information exchange in web applications.

### Key Features

- **Stateless**: Contains all necessary user information; no server-side session storage needed.
- **Integrity**: Can be signed to verify the payload's authenticity.
- **Expiration**: Supports setting an expiration time for added security.

### Structure

- **Header**: Contains token type and signing algorithm (e.g., HMAC SHA256).
- **Payload**: Contains claims (user ID, roles, expiration).
- **Signature**: Validates the token using the header, payload, and secret key.

#### JWT Format Example

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### How It Works

1. **Authentication**: Server generates a JWT after user login.
2. **Token Issuance**: Sends JWT back to the client.
3. **Token Storage**: Client stores JWT (local storage, cookies).
4. **Subsequent Requests**: Client includes JWT in `Authorization` header as a Bearer token.
5. **Token Verification**: Server checks the token's validity.

### Example Usage

#### Generating a Token

```javascript
const jwt = require("jsonwebtoken");

const createJWT = (user) => {
  return jwt.sign(
    { userId: user._id, name: user.name },
    process.env.JWT_SECRET,
    { expiresIn: "30d" } // Set token expiration
  );
};
```

#### Verifying a Token

```javascript
const verifyToken = (token) => {
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    return payload; // Decoded payload if valid
  } catch (error) {
    return null; // Token is invalid
  }
};
```

---
