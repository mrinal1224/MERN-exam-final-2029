**① Problem Statement** — student/platform description  
**② Code Stub** — what the student receives and edits  
**③ Solution Code** — the same stub, fully filled in for faculty/reference


---

# Question 1 — Create Product API

## 1. Problem Statement

You are building the backend for an e-commerce application.

The admin dashboard allows administrators to add products to the catalogue. Your task is to implement an API that receives product information, validates it, stores the product in MongoDB, and returns the newly created product.

### API

```http
POST /api/products
```

### Request Body

```json
{
    "name": "Mechanical Keyboard",
    "description": "RGB mechanical keyboard",
    "price": 4500,
    "category": "Electronics",
    "stock": 20
}
```

### Validation Rules

`name`, `price`, and `category` are required.

`price` must be a number greater than `0`.

`stock` is optional. If it is not supplied, MongoDB should use the model default of `0`. If supplied, it cannot be negative.

### Successful Response

Status:

```http
201 Created
```

```json
{
    "success": true,
    "message": "Product created successfully",
    "product": {
        "_id": "PRODUCT_ID",
        "name": "Mechanical Keyboard",
        "description": "RGB mechanical keyboard",
        "price": 4500,
        "category": "Electronics",
        "stock": 20
    }
}
```

### Errors to Handle

| Situation | Status |
|---|---:|
| Missing name | 400 |
| Missing price | 400 |
| Missing category | 400 |
| Price ≤ 0 | 400 |
| Stock < 0 | 400 |
| Database/server error | 500 |

### Hints

The product information comes from `req.body`.

Validate the request **before** creating anything in MongoDB.

Be careful while checking numeric fields because `0` is a valid JavaScript value but is falsy.

---

## 2. Code Stub — Student Version

```js
import Product from "../models/product.model.js";

export const createProduct = async (req, res) => {
    try {

        // STEP 1:
        // Extract name, description, price,
        // category and stock from req.body



        // STEP 2:
        // Validate required fields:
        // name, price and category



        // STEP 3:
        // Validate that price is greater than 0



        // STEP 4:
        // If stock is provided,
        // make sure it is not negative



        // STEP 5:
        // Create the product in MongoDB



        // STEP 6:
        // Return status 201 with the created product



    } catch (error) {

        // STEP 7:
        // Return status 500

    }
};
```

---

## 3. Solution Code

```js
import Product from "../models/product.model.js";

export const createProduct = async (req, res) => {
    try {
        const {
            name,
            description,
            price,
            category,
            stock
        } = req.body;

        if (!name || price === undefined || !category) {
            return res.status(400).json({
                success: false,
                message: "Name, price and category are required"
            });
        }

        if (typeof price !== "number" || price <= 0) {
            return res.status(400).json({
                success: false,
                message: "Price must be greater than 0"
            });
        }

        if (
            stock !== undefined &&
            (typeof stock !== "number" || stock < 0)
        ) {
            return res.status(400).json({
                success: false,
                message: "Stock cannot be negative"
            });
        }

        const product = await Product.create({
            name,
            description,
            price,
            category,
            ...(stock !== undefined && { stock })
        });

        return res.status(201).json({
            success: true,
            message: "Product created successfully",
            product
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

# Question 2 — Search and Filter Products

## 1. Problem Statement

Your e-commerce application now contains hundreds of products.

Instead of creating different endpoints for every filter, build a single API capable of filtering and sorting products based on query parameters.

### API

```http
GET /api/products
```

The following queries should work:

```http
GET /api/products

GET /api/products?category=Electronics

GET /api/products?maxPrice=5000

GET /api/products?search=keyboard

GET /api/products?category=Electronics&maxPrice=5000

GET /api/products?sort=price_asc

GET /api/products?sort=price_desc
```

### Supported Query Parameters

| Parameter | Purpose |
|---|---|
| `category` | Match a category |
| `maxPrice` | Products priced at or below this value |
| `search` | Case-insensitive product-name search |
| `sort` | `price_asc` or `price_desc` |

All filters are optional.

If no query parameters are supplied, return all products.

### Successful Response

```json
{
    "success": true,
    "count": 2,
    "products": [
        {
            "_id": "...",
            "name": "Mechanical Keyboard",
            "price": 4500,
            "category": "Electronics"
        }
    ]
}
```

If nothing matches:

```json
{
    "success": true,
    "count": 0,
    "products": []
}
```

### Errors

| Situation | Status |
|---|---:|
| `maxPrice=abc` | 400 |
| Negative maxPrice | 400 |
| Invalid sort value | 400 |
| Database error | 500 |

### Hints

All filtering information is available in `req.query`.

Start with an empty filter object and progressively add conditions.

You do not need separate database queries for different combinations of filters.

---

## 2. Code Stub — Student Version

```js
import Product from "../models/product.model.js";

export const getProducts = async (req, res) => {
    try {

        const {
            category,
            maxPrice,
            search,
            sort
        } = req.query;


        // STEP 1:
        // Create an empty MongoDB filter object



        // STEP 2:
        // If category exists, add it to the filter



        // STEP 3:
        // Validate maxPrice and add the appropriate
        // price condition



        // STEP 4:
        // If search exists, perform a
        // case-insensitive name search



        // STEP 5:
        // Validate the sort value



        // STEP 6:
        // Build the Mongoose query



        // STEP 7:
        // Apply price sorting if requested



        // STEP 8:
        // Execute the query



        // STEP 9:
        // Return products and count



    } catch (error) {

        // Handle server error

    }
};
```

---

## 3. Solution Code

```js
import Product from "../models/product.model.js";

export const getProducts = async (req, res) => {
    try {
        const {
            category,
            maxPrice,
            search,
            sort
        } = req.query;

        const filter = {};

        if (category) {
            filter.category = category;
        }

        if (maxPrice !== undefined) {
            const price = Number(maxPrice);

            if (!Number.isFinite(price) || price < 0) {
                return res.status(400).json({
                    success: false,
                    message: "Invalid maxPrice"
                });
            }

            filter.price = {
                $lte: price
            };
        }

        if (search) {
            filter.name = {
                $regex: search,
                $options: "i"
            };
        }

        const allowedSortValues = [
            "price_asc",
            "price_desc"
        ];

        if (
            sort &&
            !allowedSortValues.includes(sort)
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid sort value"
            });
        }

        let query = Product.find(filter);

        if (sort === "price_asc") {
            query = query.sort({
                price: 1
            });
        }

        if (sort === "price_desc") {
            query = query.sort({
                price: -1
            });
        }

        const products = await query;

        return res.status(200).json({
            success: true,
            count: products.length,
            products
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

# Question 3 — Update Logged-In User Profile Securely

## 1. Problem Statement

Your application allows logged-in users to update their profile.

Users are allowed to modify only:

```text
name
bio
profilePicture
```

They must **not** be able to modify:

```text
email
password
role
```

This restriction must be enforced by the backend.

For example, this request must be rejected:

```json
{
    "name": "Rahul",
    "role": "admin"
}
```

Assume authentication middleware has already populated:

```js
req.user = {
    _id: "LOGGED_IN_USER_ID"
};
```

### API

```http
PATCH /api/users/me
```

### Valid Request

```json
{
    "name": "Rahul Sharma",
    "bio": "Full Stack Developer"
}
```

### Success Response

```json
{
    "success": true,
    "message": "Profile updated successfully",
    "user": {
        "_id": "...",
        "name": "Rahul Sharma",
        "email": "rahul@example.com",
        "bio": "Full Stack Developer",
        "role": "user"
    }
}
```

The password must never be returned.

### Errors

| Situation | Status |
|---|---:|
| No authentication | 401 |
| User doesn't exist | 404 |
| Attempts to update email | 400 |
| Attempts to update password | 400 |
| Attempts to update role | 400 |
| Empty update | 400 |
| Server error | 500 |

### Hints

Never pass the complete `req.body` directly into a database update.

Think in terms of an **allow-list** of fields.

The user ID must come from authentication, not the request body.

---

## 2. Code Stub — Student Version

```js
import User from "../models/user.model.js";

export const updateProfile = async (req, res) => {
    try {

        const userId = req.user._id;

        const allowedFields = [
            "name",
            "bio",
            "profilePicture"
        ];


        // STEP 1:
        // Find which fields the client sent



        // STEP 2:
        // Check whether any field is forbidden



        // STEP 3:
        // Return 400 if a forbidden field exists



        // STEP 4:
        // Build a safe update object containing
        // only allowed fields



        // STEP 5:
        // Reject an empty update



        // STEP 6:
        // Update the authenticated user



        // STEP 7:
        // Handle user not found



        // STEP 8:
        // Return updated user without password



    } catch (error) {

        // Handle error

    }
};
```

---

## 3. Solution Code

```js
import User from "../models/user.model.js";

export const updateProfile = async (req, res) => {
    try {
        const userId = req.user._id;

        const allowedFields = [
            "name",
            "bio",
            "profilePicture"
        ];

        const incomingFields =
            Object.keys(req.body);

        const hasForbiddenField =
            incomingFields.some(
                field =>
                    !allowedFields.includes(field)
            );

        if (hasForbiddenField) {
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
                "Profile updated successfully",
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

# Question 4 — Follow / Unfollow User

## 1. Problem Statement

You are implementing the follow system for a social media application.

Each user has two arrays:

```js
followers: []
following: []
```

Suppose User A follows User B.

The backend must make both changes:

```text
A.following → add B

B.followers → add A
```

Calling the same API again should unfollow:

```text
A.following → remove B

B.followers → remove A
```

### API

```http
POST /api/users/:userId/follow
```

Authenticated user:

```js
req.user._id
```

Target user:

```js
req.params.userId
```

No body is required.

### Follow Response

```json
{
    "success": true,
    "following": true,
    "message": "User followed successfully"
}
```

### Unfollow Response

```json
{
    "success": true,
    "following": false,
    "message": "User unfollowed successfully"
}
```

### Errors

| Situation | Status |
|---|---:|
| Missing authentication | 401 |
| Invalid ObjectId | 400 |
| Follow yourself | 400 |
| User not found | 404 |
| Database failure | 500 |

### Important Requirement

Both documents must remain synchronized.

Duplicate IDs must not be created.

### Hints

MongoDB ObjectIds should not be compared as ordinary JavaScript objects.

Determine whether the relationship already exists before deciding whether to follow or unfollow.

---

## 2. Code Stub — Student Version

```js
import mongoose from "mongoose";
import User from "../models/user.model.js";

export const toggleFollowUser = async (req, res) => {
    try {

        const currentUserId = req.user._id;
        const targetUserId = req.params.userId;


        // STEP 1:
        // Validate both IDs



        // STEP 2:
        // Prevent self-follow



        // STEP 3:
        // Find current user



        // STEP 4:
        // Find target user



        // STEP 5:
        // Handle missing users



        // STEP 6:
        // Check if current user already
        // follows target user



        if (/* already following */) {

            // STEP 7:
            // Remove target from following


            // STEP 8:
            // Remove current user from followers


            // STEP 9:
            // Save both users


            // STEP 10:
            // Return unfollow response

        } else {

            // STEP 11:
            // Add target to following


            // STEP 12:
            // Add current user to followers


            // STEP 13:
            // Save both users


            // STEP 14:
            // Return follow response

        }

    } catch (error) {

        // Handle error

    }
};
```

---

## 3. Solution Code

```js
import mongoose from "mongoose";
import User from "../models/user.model.js";

export const toggleFollowUser = async (req, res) => {
    try {
        const currentUserId = req.user._id;
        const targetUserId = req.params.userId;

        if (
            !mongoose.isValidObjectId(currentUserId) ||
            !mongoose.isValidObjectId(targetUserId)
        ) {
            return res.status(400).json({
                success: false,
                message: "Invalid user id"
            });
        }

        if (
            String(currentUserId) ===
            String(targetUserId)
        ) {
            return res.status(400).json({
                success: false,
                message: "You cannot follow yourself"
            });
        }

        const currentUser =
            await User.findById(currentUserId);

        const targetUser =
            await User.findById(targetUserId);

        if (!currentUser || !targetUser) {
            return res.status(404).json({
                success: false,
                message: "User not found"
            });
        }

        const alreadyFollowing =
            currentUser.following.some(
                id => id.equals(targetUser._id)
            );

        if (alreadyFollowing) {

            currentUser.following =
                currentUser.following.filter(
                    id =>
                        !id.equals(targetUser._id)
                );

            targetUser.followers =
                targetUser.followers.filter(
                    id =>
                        !id.equals(currentUser._id)
                );

            await currentUser.save();
            await targetUser.save();

            return res.status(200).json({
                success: true,
                following: false,
                message:
                    "User unfollowed successfully"
            });
        }

        currentUser.following.push(
            targetUser._id
        );

        targetUser.followers.push(
            currentUser._id
        );

        await currentUser.save();
        await targetUser.save();

        return res.status(200).json({
            success: true,
            following: true,
            message: "User followed successfully"
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

# Question 5 — JWT Login and Authentication Middleware

## 1. Problem Statement

You are building authentication for a MERN application.

Passwords are already stored as bcrypt hashes.

Implement a login system where successful authentication generates a JWT and stores it inside an **HTTP-only cookie**.

You must also implement middleware that protects private routes.

The complete flow should be:

```text
POST /login
      ↓
Find User
      ↓
Compare Password
      ↓
Generate JWT
      ↓
Set HTTP-only Cookie
      ↓

Later request

GET /users/me
      ↓
Read Cookie
      ↓
Verify JWT
      ↓
Find User
      ↓
req.user
      ↓
Controller
```

Assume:

```env
JWT_SECRET=my_super_secret
```

### Login API

```http
POST /api/auth/login
```

Request:

```json
{
    "email": "rahul@example.com",
    "password": "secret123"
}
```

### Successful Login

```json
{
    "success": true,
    "message": "Login successful",
    "user": {
        "_id": "...",
        "name": "Rahul",
        "email": "rahul@example.com"
    }
}
```

The response must set:

```text
token=<JWT>
HttpOnly=true
```

### Protected API

```http
GET /api/users/me
```

Success:

```json
{
    "success": true,
    "user": {
        "_id": "...",
        "name": "Rahul",
        "email": "rahul@example.com"
    }
}
```

### Errors

| Situation | Status |
|---|---:|
| Missing email/password | 400 |
| Invalid credentials | 401 |
| Missing token | 401 |
| Invalid JWT | 401 |
| Expired JWT | 401 |
| JWT user no longer exists | 401 |
| Unexpected login error | 500 |

---

## 2. Code Stub — Student Version

### Login Controller

```js
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const login = async (req, res) => {
    try {

        const {
            email,
            password
        } = req.body;


        // STEP 1:
        // Validate email and password



        // STEP 2:
        // Find user using email



        // STEP 3:
        // Handle user not found



        // STEP 4:
        // Compare password using bcrypt



        // STEP 5:
        // Handle incorrect password



        // STEP 6:
        // Generate JWT



        // STEP 7:
        // Store JWT inside HTTP-only cookie



        // STEP 8:
        // Return safe user information



    } catch (error) {

        // Handle error

    }
};
```

### Authentication Middleware

```js
import jwt from "jsonwebtoken";
import User from "../models/user.model.js";

export const authenticate = async (
    req,
    res,
    next
) => {
    try {

        // STEP 1:
        // Read token from req.cookies



        // STEP 2:
        // Handle missing token



        // STEP 3:
        // Verify JWT



        // STEP 4:
        // Find user from JWT payload



        // STEP 5:
        // Exclude password



        // STEP 6:
        // Handle user not found



        // STEP 7:
        // Store user inside req.user



        // STEP 8:
        // Continue request



    } catch (error) {

        // Handle invalid or expired JWT

    }
};
```

### Get Current User

```js
export const getMe = async (req, res) => {

    // TODO:
    // Return the authenticated user
    // available in req.user

};
```

### Route

```js
import express from "express";

import {
    authenticate
} from "../middlewares/auth.middleware.js";

import {
    getMe
} from "../controllers/user.controller.js";

const router = express.Router();

router.get(
    "/me",

    // TODO:
    // Add middleware and controller

);

export default router;
```

---

## 3. Solution Code

### `auth.controller.js`

```js
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

import User from "../models/user.model.js";

export const login = async (req, res) => {
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

        const user =
            await User.findOne({
                email
            });

        if (!user) {
            return res.status(401).json({
                success: false,
                message: "Invalid credentials"
            });
        }

        const passwordMatches =
            await bcrypt.compare(
                password,
                user.password
            );

        if (!passwordMatches) {
            return res.status(401).json({
                success: false,
                message: "Invalid credentials"
            });
        }

        const token = jwt.sign(
            {
                userId: user._id
            },
            process.env.JWT_SECRET,
            {
                expiresIn: "1h"
            }
        );

        res.cookie(
            "token",
            token,
            {
                httpOnly: true,
                maxAge: 60 * 60 * 1000
            }
        );

        return res.status(200).json({
            success: true,
            message: "Login successful",
            user: {
                _id: user._id,
                name: user.name,
                email: user.email
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

### `auth.middleware.js`

```js
import jwt from "jsonwebtoken";
import User from "../models/user.model.js";

export const authenticate = async (
    req,
    res,
    next
) => {
    try {
        const token =
            req.cookies?.token;

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

        const user =
            await User
                .findById(decoded.userId)
                .select("-password");

        if (!user) {
            return res.status(401).json({
                success: false,
                message: "Authentication required"
            });
        }

        req.user = user;

        next();

    } catch (error) {
        return res.status(401).json({
            success: false,
            message: "Authentication failed"
        });
    }
};
```

### `user.controller.js`

```js
export const getMe = async (req, res) => {
    return res.status(200).json({
        success: true,
        user: {
            _id: req.user._id,
            name: req.user.name,
            email: req.user.email
        }
    });
};
```

### `user.routes.js`

```js
import express from "express";

import {
    authenticate
} from "../middlewares/auth.middleware.js";

import {
    getMe
} from "../controllers/user.controller.js";

const router = express.Router();

router.get(
    "/me",
    authenticate,
    getMe
);

export default router;
```

