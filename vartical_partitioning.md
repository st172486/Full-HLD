# Vertical Partitioning - Detailed Notes

## 📌 What is Vertical Partitioning?

Vertical partitioning is a database design technique where a table is
split **column-wise** into multiple tables.

------------------------------------------------------------------------

## 🧠 Example

### Original Table

| user_id \| name \| email \| password \| profile_pic \| bio \|
  last_login \|

### After Partitioning

#### Users_Basic

| user_id \| name \| email \| last_login \|

#### Users_Profile

| user_id \| profile_pic \| bio \|

#### Users_Auth

| user_id \| password \|

------------------------------------------------------------------------

## 📊 Diagram

              Users Table
    -----------------------------------
    | id | name | email | password |
    | pic | bio | last_login        |
    -----------------------------------

                ↓ Split

       ---------------------     ---------------------
       | Users_Basic        |     | Users_Profile     |
       | id, name, email   |     | id, pic, bio     |
       ---------------------     ---------------------

                 ---------------------
                 | Users_Auth        |
                 | id, password      |
                 ---------------------

------------------------------------------------------------------------

## 🚀 Advantages

### 1. Performance Improvement

-   Fetch only required columns
-   Faster queries

### 2. Reduced I/O

-   Less data read from disk

### 3. Better Security

-   Sensitive data isolated

### 4. Better Caching

-   Frequently used data cached separately

### 5. Independent Scaling

-   Move heavy columns to separate storage

------------------------------------------------------------------------

## ⚠️ Trade-offs

### 1. Joins Required

Need JOINs to fetch complete data

### 2. Complexity

More tables and queries

------------------------------------------------------------------------

## 🆚 Vertical vs Horizontal Partitioning

  Feature   Vertical           Horizontal
  --------- ------------------ ------------------
  Split     Columns            Rows
  Use       Optimize queries   Scale large data

------------------------------------------------------------------------

## 🏗️ Real-World Use Cases

-   Social Media (Auth vs Profile)
-   E-commerce (Basic vs Details)
-   Large systems (Move images to S3)

------------------------------------------------------------------------

## 🎯 Interview Tip

Use vertical partitioning when: - Table has many columns - Queries use
only subset of columns - Need to isolate sensitive data

------------------------------------------------------------------------

## 🔥 Summary

Vertical partitioning splits tables by columns to improve performance,
security, and scalability.
