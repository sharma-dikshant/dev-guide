# Error Handling in Express

## Types of Errors:

- **Operational Error:** Errors that are expected like validation errors, etc.
- **Programming Error:** Errors that are not expected like syntax errors, programming error, etc.

## Error Handling Features in Express:

**1:** Anytime you call `next(error)` from anywhere in your application,
Express will skip all remaining middleware and directly go to the error handling middleware.  
**2:** You can define your own **error handling middleware** by adding a middleware with 4 parameters. `(err, req, res, next)`.

**3.** `app.all('*', (req, res, next) => { ... })` will catch all requests that are not handled by any route. From there, you can call `next(new Error('Not Found'))` to pass the error to the error handling middleware.

## Steps to Handle Errors in Express:
### 1. Creating a Custom Error Class:
```javascript
// utils/AppError.js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message); // Call parent constructor

    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    this.isOperational = true;

    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = AppError;

```

### 2. Use a wraper function to handle async errors:
```javascript
// utils/catchAsync.js
module.exports = (fn) => {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
};

```

### 3. Create a global error handling middleware:
this is the heart of the entire error handling process.  
```javascript
// middlewares/errorHandler.js
const sendErrorDev = (err, res) => {
  res.status(err.statusCode || 500).json({
    status: err.status || 'error',
    message: err.message,
    stack: err.stack,
    error: err,
  });
};

const sendErrorProd = (err, res) => {
  // Operational, trusted error
  if (err.isOperational) {
    res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
    });
  } else {
    // Programming/unknown error
    console.error('ERROR 💥:', err);
    res.status(500).json({
      status: 'error',
      message: 'Something went very wrong!',
    });
  }
};

module.exports = (err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (process.env.NODE_ENV === 'development') {
    sendErrorDev(err, res);
  } else {
    sendErrorProd(err, res);
  }
};

```

### 4. Use the error handling middleware in your app:
```javascript
// your routes here...

app.all('*', (req, res, next) => {
  const AppError = require('./utils/AppError');
  next(new AppError(`Can't find ${req.originalUrl} on this server`, 404));
});

// make sure error handling middleware is the last middleware in your app
// Global error handler
app.use(errorHandler);
```

### 5. Use the custom error class in controllers:
```javascript
const catchAsync = require('../utils/catchAsync');
const AppError = require('../utils/AppError');

exports.getUser = catchAsync(async (req, res, next) => {
  const user = await User.findById(req.params.id);

  if (!user) return next(new AppError('No user found with this ID', 404));

  res.status(200).json({
    status: 'success',
    data: { user },
  });
});

```

### 6. Handling uncaught exceptions and unhandled rejections:
```javascript
// server.js
process.on('uncaughtException', err => {
  console.error('UNCAUGHT EXCEPTION 💥 Shutting down...');
  console.error(err.name, err.message);
  process.exit(1);
});

process.on('unhandledRejection', err => {
  console.error('UNHANDLED REJECTION 💥 Shutting down...');
  console.error(err.name, err.message);
  server.close(() => process.exit(1));
});

```