# SET B — MERN Backend

## Question 1 — Create an Event API

### 1. Problem Statement

You are building the backend for an event management application.

Organizers can create events that users can later browse and register for.

Implement an API that validates the event information and stores a new event in MongoDB.

### Existing Event Model

```js
import mongoose from "mongoose";

const eventSchema = new mongoose.Schema(
    {
        title: {
            type: String,
            required: true
        },

        description: {
            type: String,
            default: ""
        },

        location: {
            type: String,
            required: true
        },

        ticketPrice: {
            type: Number,
            required: true
        },

        capacity: {
            type: Number,
            default: 50
        }
    },
    {
        timestamps: true
    }
);

export default mongoose.model(
    "Event",
    eventSchema
);
```

### API

```http
POST /api/events
```

### Request Body

```json
{
    "title": "JavaScript Meetup",
    "description": "A meetup for JS developers",
    "location": "Bengaluru",
    "ticketPrice": 499,
    "capacity": 100
}
```

Required:

```text
title
location
ticketPrice
```

`capacity` is optional and defaults to `50`.

### Validation Rules

`title` and `location` cannot be empty.

`ticketPrice` must be a number greater than or equal to `0`.

`capacity`, if supplied, must be a positive number.

### Successful Response

Status `201`.

```json
{
    "success": true,
    "message": "Event created successfully",
    "event": {
        "_id": "...",
        "title": "JavaScript Meetup",
        "location": "Bengaluru",
        "ticketPrice": 499,
        "capacity": 100
    }
}
```

### Errors

| Situation | Status |
|---|---:|
| Missing title | 400 |
| Missing location | 400 |
| Missing ticketPrice | 400 |
| Negative ticketPrice | 400 |
| Capacity ≤ 0 | 400 |
| Server/DB failure | 500 |

### Hints

Validate the request before calling Mongoose.

Remember that `ticketPrice = 0` represents a free event and is valid.

---

### 2. Code Stub

```js
import Event from "../models/event.model.js";

export const createEvent = async (req, res) => {
    try {

        // STEP 1:
        // Extract event information from req.body


        // STEP 2:
        // Validate required fields


        // STEP 3:
        // Validate ticketPrice


        // STEP 4:
        // Validate capacity if supplied


        // STEP 5:
        // Create event using Mongoose


        // STEP 6:
        // Return 201 response


    } catch (error) {

        // Handle unexpected error

    }
};
```

---

### 3. Solution Code

```js
import Event from "../models/event.model.js";

export const createEvent = async (req, res) => {
    try {
        const {
            title,
            description,
            location,
            ticketPrice,
            capacity
        } = req.body;

        if (
            !title ||
            !location ||
            ticketPrice === undefined
        ) {
            return res.status(400).json({
                success: false,
                message:
                    "Title, location and ticketPrice are required"
            });
        }

        if (
            typeof ticketPrice !== "number" ||
            ticketPrice < 0
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid ticket price"
            });
        }

        if (
            capacity !== undefined &&
            (
                typeof capacity !== "number" ||
                capacity <= 0
            )
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid capacity"
            });
        }

        const event = await Event.create({
            title,
            description,
            location,
            ticketPrice,
            ...(capacity !== undefined && {
                capacity
            })
        });

        return res.status(201).json({
            success: true,
            message: "Event created successfully",
            event
        });

    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Internal server error"
        });
    }
};
```

---

# Question 2 — Search and Filter Jobs

### 1. Problem Statement

You are building the backend for a job portal.

The frontend needs a single endpoint that can list jobs while optionally filtering them by:

```text
location
job type
minimum salary
job title
```

The results should also optionally be sortable by salary.

### API

```http
GET /api/jobs
```

Examples:

```http
GET /api/jobs

GET /api/jobs?location=Bengaluru

GET /api/jobs?type=remote

GET /api/jobs?minSalary=800000

GET /api/jobs?search=frontend

GET /api/jobs?sort=salary_desc

GET /api/jobs?location=Bengaluru&type=remote&minSalary=800000
```

### Job Structure

```js
{
    title: String,
    company: String,
    location: String,
    type: String,
    salary: Number
}
```

Supported `sort` values:

```text
salary_asc
salary_desc
```

Search should perform a **case-insensitive match against job title**.

All filters are optional.

### Response

```json
{
    "success": true,
    "count": 2,
    "jobs": []
}
```

No matches should return:

```json
{
    "success": true,
    "count": 0,
    "jobs": []
}
```

### Errors

| Situation | Status |
|---|---:|
| Invalid minSalary | 400 |
| Negative minSalary | 400 |
| Unsupported sort | 400 |
| Database failure | 500 |

### Hints

Construct the MongoDB filter dynamically from `req.query`.

For minimum salary, think about which MongoDB comparison operator you need.

---

### 2. Code Stub

```js
import Job from "../models/job.model.js";

export const getJobs = async (req, res) => {
    try {

        const {
            location,
            type,
            minSalary,
            search,
            sort
        } = req.query;


        // STEP 1:
        // Create filter object


        // STEP 2:
        // Add location filter


        // STEP 3:
        // Add type filter


        // STEP 4:
        // Validate and add minimum salary


        // STEP 5:
        // Add case-insensitive title search


        // STEP 6:
        // Validate sort


        // STEP 7:
        // Build Mongoose query


        // STEP 8:
        // Apply salary sorting


        // STEP 9:
        // Execute query and return result


    } catch (error) {

        // Handle error

    }
};
```

---

### 3. Solution Code

```js
import Job from "../models/job.model.js";

export const getJobs = async (req, res) => {
    try {
        const {
            location,
            type,
            minSalary,
            search,
            sort
        } = req.query;

        const filter = {};

        if (location) {
            filter.location = location;
        }

        if (type) {
            filter.type = type;
        }

        if (minSalary !== undefined) {
            const salary = Number(minSalary);

            if (
                !Number.isFinite(salary) ||
                salary < 0
            ) {
                return res.status(400).json({
                    success: false,
                    message: "Invalid minSalary"
                });
            }

            filter.salary = {
                $gte: salary
            };
        }

        if (search) {
            filter.title = {
                $regex: search,
                $options: "i"
            };
        }

        if (
            sort &&
            ![
                "salary_asc",
                "salary_desc"
            ].includes(sort)
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid sort value"
            });
        }

        let query = Job.find(filter);

        if (sort === "salary_asc") {
            query = query.sort({
                salary: 1
            });
        }

        if (sort === "salary_desc") {
            query = query.sort({
                salary: -1
            });
        }

        const jobs = await query;

        return res.status(200).json({
            success: true,
            count: jobs.length,
            jobs
        });

    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Internal server error"
        });
    }
};
```

---

# Question 3 — Secure Account Settings Update

### 1. Problem Statement

You are building an account settings page.

Authenticated users can modify:

```text
displayName
bio
city
```

However, users must not be allowed to modify security or account-management fields such as:

```text
email
password
accountStatus
```

Assume authentication middleware has already populated:

```js
req.user = {
    _id: "LOGGED_IN_USER_ID"
};
```

### API

```http
PATCH /api/account/settings
```

Valid request:

```json
{
    "displayName": "Rahul S",
    "bio": "Software Engineer",
    "city": "Bengaluru"
}
```

Malicious request:

```json
{
    "displayName": "Rahul",
    "accountStatus": "verified"
}
```

The second request must be rejected completely.

### Success Response

```json
{
    "success": true,
    "message": "Account settings updated",
    "user": {
        "_id": "...",
        "displayName": "Rahul S",
        "email": "rahul@example.com",
        "bio": "Software Engineer",
        "city": "Bengaluru"
    }
}
```

Password must not appear.

### Errors

| Situation | Status |
|---|---:|
| Forbidden field | 400 |
| Empty update | 400 |
| User doesn't exist | 404 |
| Server error | 500 |

### Hints

Do not trust the keys received from `req.body`.

Create a whitelist of fields the user is allowed to modify.

---

### 2. Code Stub

```js
import User from "../models/user.model.js";

export const updateAccountSettings = async (
    req,
    res
) => {
    try {

        const userId = req.user._id;

        const allowedFields = [
            "displayName",
            "bio",
            "city"
        ];


        // STEP 1:
        // Get fields sent by client


        // STEP 2:
        // Detect forbidden fields


        // STEP 3:
        // Reject request if any forbidden
        // field was supplied


        // STEP 4:
        // Build safe update object


        // STEP 5:
        // Reject empty update


        // STEP 6:
        // Update user


        // STEP 7:
        // Exclude password


        // STEP 8:
        // Handle user not found


        // STEP 9:
        // Return response


    } catch (error) {

        // Handle error

    }
};
```

---

### 3. Solution Code

```js
import User from "../models/user.model.js";

export const updateAccountSettings = async (
    req,
    res
) => {
    try {
        const userId = req.user._id;

        const allowedFields = [
            "displayName",
            "bio",
            "city"
        ];

        const incomingFields =
            Object.keys(req.body);

        const containsForbiddenField =
            incomingFields.some(
                field =>
                    !allowedFields.includes(field)
            );

        if (containsForbiddenField) {
            return res.status(400).json({
                success: false,
                message:
                    "One or more fields cannot be updated"
            });
        }

        const updateData = {};

        for (const field of allowedFields) {
            if (req.body[field] !== undefined) {
                updateData[field] =
                    req.body[field];
            }
        }

        if (
            Object.keys(updateData).length === 0
        ) {
            return res.status(400).json({
                success: false,
                message: "No valid fields supplied"
            });
        }

        const user =
            await User
                .findByIdAndUpdate(
                    userId,
                    updateData,
                    {
                        new: true,
                        runValidators: true
                    }
                )
                .select("-password");

        if (!user) {
            return res.status(404).json({
                success: false,
                message: "User not found"
            });
        }

        return res.status(200).json({
            success: true,
            message:
                "Account settings updated",
            user
        });

    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Internal server error"
        });
    }
};
```

---

# Question 4 — Add / Remove Product from Wishlist

### 1. Problem Statement

You are building the wishlist feature for an e-commerce application.

Each user has:

```js
wishlist: [
    {
        type: mongoose.Schema.Types.ObjectId,
        ref: "Product"
    }
]
```

Build a single toggle API.

If the product is not currently in the user's wishlist:

```text
Add it
```

If it already exists:

```text
Remove it
```

### API

```http
POST /api/wishlist/:productId
```

The logged-in user is available as:

```js
req.user._id
```

The product comes from:

```js
req.params.productId
```

### Add Response

```json
{
    "success": true,
    "inWishlist": true,
    "message": "Product added to wishlist"
}
```

### Remove Response

```json
{
    "success": true,
    "inWishlist": false,
    "message": "Product removed from wishlist"
}
```

### Requirements

Before modifying the wishlist:

- Validate `productId`.
- Ensure the product actually exists.
- Ensure the user exists.
- Never create duplicate product IDs.

### Errors

| Situation | Status |
|---|---:|
| Invalid product ID | 400 |
| Product not found | 404 |
| User not found | 404 |
| Database failure | 500 |

### Hints

The main challenge is determining whether the product's ObjectId already exists in the wishlist array.

Remember that Mongoose ObjectIds require careful comparison.

---

### 2. Code Stub

```js
import mongoose from "mongoose";

import User from "../models/user.model.js";
import Product from "../models/product.model.js";

export const toggleWishlist = async (
    req,
    res
) => {
    try {

        const userId = req.user._id;
        const productId =
            req.params.productId;


        // STEP 1:
        // Validate productId


        // STEP 2:
        // Find product


        // STEP 3:
        // Handle product not found


        // STEP 4:
        // Find authenticated user


        // STEP 5:
        // Handle user not found


        // STEP 6:
        // Check if product already exists
        // inside wishlist


        if (/* product already exists */) {

            // STEP 7:
            // Remove product from wishlist


            // STEP 8:
            // Save user


            // STEP 9:
            // Return remove response

        } else {

            // STEP 10:
            // Add product to wishlist


            // STEP 11:
            // Save user


            // STEP 12:
            // Return add response

        }

    } catch (error) {

        // Handle error

    }
};
```

---

### 3. Solution Code

```js
import mongoose from "mongoose";

import User from "../models/user.model.js";
import Product from "../models/product.model.js";

export const toggleWishlist = async (
    req,
    res
) => {
    try {
        const userId = req.user._id;
        const productId =
            req.params.productId;

        if (
            !mongoose.isValidObjectId(productId)
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid product id"
            });
        }

        const product =
            await Product.findById(productId);

        if (!product) {
            return res.status(404).json({
                success: false,
                message: "Product not found"
            });
        }

        const user =
            await User.findById(userId);

        if (!user) {
            return res.status(404).json({
                success: false,
                message: "User not found"
            });
        }

        const alreadyInWishlist =
            user.wishlist.some(
                id => id.equals(product._id)
            );

        if (alreadyInWishlist) {

            user.wishlist =
                user.wishlist.filter(
                    id =>
                        !id.equals(product._id)
                );

            await user.save();

            return res.status(200).json({
                success: true,
                inWishlist: false,
                message:
                    "Product removed from wishlist"
            });
        }

        user.wishlist.push(product._id);

        await user.save();

        return res.status(200).json({
            success: true,
            inWishlist: true,
            message:
                "Product added to wishlist"
        });

    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Internal server error"
        });
    }
};
```

---

# Question 5 — JWT Authentication for Admin Routes

### 1. Problem Statement

You are building an admin dashboard for an application.

Administrators log in using their email and password.

After successful login:

```text
Validate credentials
        ↓
Generate JWT
        ↓
Store JWT in HTTP-only cookie
        ↓
Future requests send cookie
        ↓
Middleware verifies JWT
        ↓
Admin becomes available in req.admin
```

Passwords are already stored as bcrypt hashes.

### Admin Model

```js
{
    name: String,
    email: String,
    password: String,
    role: String
}
```

Only users whose:

```js
role === "admin"
```

may log in through this endpoint.

### Login API

```http
POST /api/admin/login
```

Request:

```json
{
    "email": "admin@example.com",
    "password": "admin123"
}
```

### Success

```json
{
    "success": true,
    "message": "Admin login successful",
    "admin": {
        "_id": "...",
        "name": "Admin",
        "email": "admin@example.com"
    }
}
```

Set an HTTP-only cookie named:

```text
adminToken
```

### Protected API

```http
GET /api/admin/me
```

The authentication middleware must:

```text
Read adminToken
Verify JWT
Find admin
Check role
Attach admin to req.admin
Call next()
```

### Errors

| Situation | Status |
|---|---:|
| Missing email/password | 400 |
| Invalid credentials | 401 |
| User is not admin | 401 |
| Missing cookie | 401 |
| Invalid/expired JWT | 401 |
| Admin deleted | 401 |
| Server failure | 500 |

---

### 2. Code Stub

#### Login Controller

```js
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const adminLogin = async (
    req,
    res
) => {
    try {

        const {
            email,
            password
        } = req.body;


        // STEP 1:
        // Validate email and password


        // STEP 2:
        // Find user by email


        // STEP 3:
        // Ensure user exists and role is admin


        // STEP 4:
        // Compare password


        // STEP 5:
        // Generate JWT


        // STEP 6:
        // Store JWT in adminToken
        // HTTP-only cookie


        // STEP 7:
        // Return safe admin information


    } catch (error) {

        // Handle error

    }
};
```

#### Admin Middleware

```js
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const authenticateAdmin = async (
    req,
    res,
    next
) => {
    try {

        // STEP 1:
        // Read adminToken cookie


        // STEP 2:
        // Handle missing token


        // STEP 3:
        // Verify JWT


        // STEP 4:
        // Find admin


        // STEP 5:
        // Make sure user still has admin role


        // STEP 6:
        // Attach user to req.admin


        // STEP 7:
        // Call next()


    } catch (error) {

        // Handle invalid/expired JWT

    }
};
```

#### Current Admin Controller

```js
export const getCurrentAdmin = async (
    req,
    res
) => {

    // TODO:
    // Return safe admin information
    // from req.admin

};
```

---

### 3. Solution Code

#### Login Controller

```js
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const adminLogin = async (
    req,
    res
) => {
    try {
        const {
            email,
            password
        } = req.body;

        if (!email || !password) {
            return res.status(400).json({
                success: false,
                message:
                    "Email and password are required"
            });
        }

        const admin =
            await User.findOne({
                email
            });

        if (
            !admin ||
            admin.role !== "admin"
        ) {
            return res.status(401).json({
                success: false,
                message: "Invalid credentials"
            });
        }

        const passwordMatches =
            await bcrypt.compare(
                password,
                admin.password
            );

        if (!passwordMatches) {
            return res.status(401).json({
                success: false,
                message: "Invalid credentials"
            });
        }

        const token = jwt.sign(
            {
                userId: admin._id
            },
            process.env.JWT_SECRET,
            {
                expiresIn: "1h"
            }
        );

        res.cookie(
            "adminToken",
            token,
            {
                httpOnly: true,
                maxAge: 60 * 60 * 1000
            }
        );

        return res.status(200).json({
            success: true,
            message:
                "Admin login successful",
            admin: {
                _id: admin._id,
                name: admin.name,
                email: admin.email
            }
        });

    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Internal server error"
        });
    }
};
```

#### Admin Authentication Middleware

```js
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const authenticateAdmin = async (
    req,
    res,
    next
) => {
    try {
        const token =
            req.cookies?.adminToken;

        if (!token) {
            return res.status(401).json({
                success: false,
                message: "Authentication required"
            });
        }

        const decoded =
            jwt.verify(
                token,
                process.env.JWT_SECRET
            );

        const admin =
            await User
                .findById(decoded.userId)
                .select("-password");

        if (
            !admin ||
            admin.role !== "admin"
        ) {
            return res.status(401).json({
                success: false,
                message: "Authentication required"
            });
        }

        req.admin = admin;

        next();

    } catch (error) {
        return res.status(401).json({
            success: false,
            message: "Authentication failed"
        });
    }
};
```

#### Current Admin Controller

```js
export const getCurrentAdmin = async (
    req,
    res
) => {
    return res.status(200).json({
        success: true,
        admin: {
            _id: req.admin._id,
            name: req.admin.name,
            email: req.admin.email
        }
    });
};
```


