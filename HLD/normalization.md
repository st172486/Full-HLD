# Database Normalization - Complete Interview Notes

> **Target Audience:** Software Engineers & System Design Interview Preparation
>
> **Goal:** Understand *why* normalization exists, *when* to use it, and *how* to think about database design instead of memorizing definitions.

---

# Table of Contents

1. What is Normalization?
2. Why Do We Need Normalization?
3. Problems Without Normalization
   - Data Duplication
   - Update Anomaly
   - Insert Anomaly
   - Delete Anomaly
4. Step-by-Step Example
5. First Normal Form (1NF)
6. Second Normal Form (2NF)
7. Third Normal Form (3NF)
8. BCNF
9. How to Think About Normalization
10. Normalization vs Denormalization
11. Real-World Examples
12. Interview Tips

---

# What is Normalization?

Database normalization is the process of organizing data into multiple related tables so that:

- Every piece of information is stored only once.
- Data redundancy is minimized.
- Data consistency is maintained.
- Database anomalies are eliminated.

## Simple Definition

> **Normalization means storing every fact exactly once.**

Instead of keeping duplicate information in multiple places, we separate different entities into different tables and connect them using relationships.

---

# Why Do We Need Normalization?

Imagine you're building an Employee Management System.

Your manager says:

> "Just store everything in one table."

You create the following table.

| EmployeeId | EmployeeName | Department | DepartmentHead | DepartmentLocation |
|------------|--------------|------------|----------------|--------------------|
|1|Suraj|Engineering|John|Hyderabad|
|2|Rahul|Engineering|John|Hyderabad|
|3|Priya|Engineering|John|Hyderabad|
|4|Amit|HR|Lisa|Bangalore|
|5|Neha|HR|Lisa|Bangalore|

Looks perfectly fine...

Until the database starts growing.

---

# Problems Without Normalization

## 1. Data Duplication

Engineering has 1000 employees.

That means these values are repeated 1000 times:

- Engineering
- John
- Hyderabad

### Result

Huge storage waste.

Instead of storing

```
Engineering
```

once,

you store it thousands of times.

---

## 2. Update Anomaly

Engineering office moves from Hyderabad to Bangalore.

You now have to update every Engineering employee.

Suppose one update fails.

|Employee|Department|Location|
|----------|------------|------------|
|Suraj|Engineering|Bangalore|
|Rahul|Engineering|Bangalore|
|Priya|Engineering|Hyderabad|

Now the database contains conflicting information.

### This is called:

> **Update Anomaly**

---

## 3. Insert Anomaly

A new department called AI Research has been created.

No employees have joined yet.

Can you insert it?

No.

Because Employee fields are mandatory.

You cannot store the department until an employee joins.

### This is called:

> **Insert Anomaly**

---

## 4. Delete Anomaly

Suppose HR has only one employee.

|Employee|Department|
|----------|------------|
|Amit|HR|

Amit resigns.

Delete the row.

Now HR department also disappears.

You accidentally lost department information.

### This is called:

> **Delete Anomaly**

---

# Why Do These Problems Happen?

Because one table stores information about multiple independent entities.

Our table contains:

- Employee Information
- Department Information

These are two different things.

Whenever multiple entities are stored together,

redundancy naturally appears.

---

# Solution: Normalize the Database

Instead of one table,

split the entities.

## Employees Table

|EmployeeId|EmployeeName|DepartmentId|
|-----------|------------|------------|
|1|Suraj|10|
|2|Rahul|10|
|3|Priya|10|
|4|Amit|20|

---

## Departments Table

|DepartmentId|Department|Head|Location|
|------------|-----------|----|---------|
|10|Engineering|John|Hyderabad|
|20|HR|Lisa|Bangalore|

Now:

- Engineering is stored once.
- Department Head is stored once.
- Location is stored once.

---

# Benefits After Normalization

## No Data Duplication

Engineering appears only once.

---

## Easy Updates

Move Engineering office?

Update one row.

Done.

---

## Easy Insert

Create department without employees.

Possible.

---

## Easy Delete

Delete an employee.

Department still exists.

---

# First Normal Form (1NF)

## Rule

Every column should contain only one value.

### Bad Example

|Employee|Skills|
|---------|-------------------|
|Suraj|Java, React, C#|

The Skills column stores multiple values.

This is not atomic.

---

### Good Design

## Employee

|Id|Name|
|--|----|
|1|Suraj|

---

## EmployeeSkills

|EmployeeId|Skill|
|-----------|------|
|1|Java|
|1|React|
|1|C#|

Every cell now contains exactly one value.

This satisfies **1NF**.

---

# Second Normal Form (2NF)

## Rule

Every non-key column must depend on the entire primary key.

2NF is only relevant when a table has a **Composite Primary Key**.

---

Example:

StudentCourse

Primary Key:

```
(StudentId, CourseId)
```

Table

|StudentId|CourseId|StudentName|CourseName|
|----------|----------|------------|-----------|

Notice:

StudentName depends only on StudentId.

CourseName depends only on CourseId.

Neither depends on the complete key.

This is called a:

> **Partial Dependency**

---

## Correct Design

Students

|StudentId|StudentName|

Courses

|CourseId|CourseName|

StudentCourses

|StudentId|CourseId|

Now every non-key attribute depends on the entire composite key.

This satisfies **2NF**.

---

# Third Normal Form (3NF)

## Rule

No Transitive Dependency.

A non-key column should depend only on the primary key.

---

Example

Employees

|EmployeeId|DepartmentId|DepartmentHead|

Question:

Does DepartmentHead depend on Employee?

No.

It depends on Department.

Relationship

```
Employee
     ↓
Department
     ↓
Department Head
```

This is called:

> **Transitive Dependency**

---

## Correct Design

Employees

|EmployeeId|DepartmentId|

Departments

|DepartmentId|DepartmentHead|

Now DepartmentHead depends directly on Department.

This satisfies **3NF**.

---

# BCNF (Boyce-Codd Normal Form)

BCNF is a stricter version of 3NF.

Rule:

> Every determinant must be a candidate key.

BCNF handles rare cases involving multiple candidate keys and overlapping functional dependencies.

For most production applications,

3NF is usually sufficient.

---

# Real World Example

## E-Commerce

Suppose Amazon stores:

|OrderId|CustomerName|CustomerAddress|ProductName|Price|

Customer places 100 orders.

Address gets repeated 100 times.

Change address?

Update 100 rows.

---

Better Design

Customers

|CustomerId|Name|Address|

Products

|ProductId|Name|Price|

Orders

|OrderId|CustomerId|

OrderItems

|OrderId|ProductId|

Everything is stored exactly once.

---

# How to Think About Normalization

Don't memorize 1NF, 2NF and 3NF.

Instead ask yourself these questions.

---

## Question 1

Am I storing the same information repeatedly?

If yes,

Normalize.

---

## Question 2

Will updating one fact require updating hundreds of rows?

If yes,

Normalize.

---

## Question 3

Can deleting one row accidentally remove unrelated information?

If yes,

Normalize.

---

## Question 4

Can I insert an entity without another unrelated entity?

If no,

Normalize.

---

# Normalization vs Denormalization

| Normalization | Denormalization |
|---------------|-----------------|
| Reduces redundancy | Introduces controlled redundancy |
| Improves consistency | Improves read performance |
| Requires joins | Reduces joins |
| Better for updates | Better for reads |
| Preferred in OLTP systems | Preferred in Analytics & Reporting |

---

# Why Don't Big Companies Fully Normalize?

Suppose you're building a Candidate Management System.

To display one candidate profile, you need:

- Candidate
- Resume
- Skills
- Education
- Experience
- Recruiter
- Organization
- Notes
- Job
- Activities

Fully normalized databases require many joins.

Joins become expensive.

Therefore,

companies intentionally duplicate some data.

Example:

Candidate table may also contain

- CurrentCompany
- CurrentDesignation
- SkillsSummary

Even though this information exists elsewhere.

This is called:

> **Denormalization**

---

# When Should You Normalize?

Normalize when:

- Designing transactional databases
- Data consistency is critical
- Storage efficiency matters
- Frequent updates happen
- Multiple users modify data

Examples

- Banking
- HR Systems
- ERP
- Hospital Management
- Recruitment Systems
- E-Commerce

---

# When Should You Denormalize?

Denormalize when:

- Read performance is critical
- Reports are generated frequently
- Data changes rarely
- Reducing joins improves response time

Examples

- Analytics
- Dashboards
- Reporting
- Elasticsearch
- Data Warehouses

---

# Common Interview Questions

### Q1. What is normalization?

Normalization is the process of organizing data into multiple related tables to eliminate redundancy and maintain consistency.

---

### Q2. Why do we normalize databases?

To eliminate:

- Data duplication
- Update anomalies
- Insert anomalies
- Delete anomalies

---

### Q3. What are database anomalies?

- Update Anomaly
- Insert Anomaly
- Delete Anomaly

---

### Q4. What is denormalization?

Denormalization intentionally duplicates data to improve read performance by reducing joins.

---

### Q5. Is normalization always good?

No.

Highly normalized databases require many joins.

For read-heavy systems,

controlled denormalization is often preferred.

---

# Interview Answer (2-Minute Version)

> Database normalization is the process of organizing data so that every fact is stored only once. The primary goal is to eliminate redundancy and maintain data consistency. Without normalization, databases suffer from update, insert, and delete anomalies. During schema design, I first identify independent entities like Users, Departments, Products, or Orders and store them in separate tables with relationships. This results in a clean, maintainable, and consistent database. However, in large-scale systems where read performance is more important than write performance, we may intentionally denormalize certain data to reduce joins and improve query speed.

---

# Key Takeaways

- Store every fact exactly once.
- Separate independent entities into different tables.
- Eliminate redundancy.
- Prevent update, insert, and delete anomalies.
- Normalize first.
- Denormalize later only if performance requires it.
- Good database design is about balancing consistency and performance.

---

# Interview Cheat Sheet

```
Normalization
      │
      ▼
Store every fact only once
      │
      ▼
Reduces Redundancy
      │
      ▼
Prevents
├── Update Anomaly
├── Insert Anomaly
└── Delete Anomaly
      │
      ▼
Normal Forms
├── 1NF → Atomic values
├── 2NF → No Partial Dependency
├── 3NF → No Transitive Dependency
└── BCNF → Every determinant is a candidate key
      │
      ▼
Better Data Consistency
      │
      ▼
More Joins
      │
      ▼
Denormalize (Only if Read Performance Requires It)
```

---

# Golden Rule (Remember Forever)

> **Normalize for correctness. Denormalize for performance.**

Or even simpler:

> **Store every fact once. Duplicate it only when you have a measured performance reason.**