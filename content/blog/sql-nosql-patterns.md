---
title: "Comparing Data Models and Query Patterns in SQL and NoSQL"
date: 2022-04-17T14:58:23-04:00
draft: true
tags: [databases, cis188-lecture, series]
description: ""
---

I was invited back to Penn last week to teach a guest lecture for [CIS 188](https://cis188.org), an elective class that covers DevOps and scaling web applications. I was a TA for 188 in the spring of 2021, so it was great to have an opportunity to get in front of the class and give one of the special topics lectures. I decided to give my lecture about the challenges and techniques with scaling distributed databases. I really enjoyed giving the lecture, and wanted to tidy up my notes to publish on the blog. The flow might feel a bit more different than a regular blog post, and every heading rougly corresponds to a slide in the original presentation.

There's no database course as a prerequisite for CIS 188, so the first part of the lecture was an overview of the database landscape today, and a high-level overview of the differences between SQL and NoSQL databases.

<!--more-->

I just want to reiterate that I'm giving my own opinions here and they don't reflect those of my employer :)

## What is a database?

Paraphrased from Oracle's website, a database is:

> An organized collection of structured, queryable information.

A filesystem might contain tons of structured JSON files, but filesystems don't have the capacity to query for data by anything except for ID. Being able to store information AND get it back out again are key components of a database.

### Examples of databases

- MySQL
- Cassandra
- Redis
  - You've used Redis throughout this class. Its homepage describes it as an "in-memory data store" that can be used "as a database, cache, streaming engine and message broker"

### What is SQL?

Since the 1980s, the dominant database management systems (DBMSes) have followed the relational model and over time have standardized on the Structured Query Language, which goes by its acronym SQL (pronounced like the word "sequel"). SQL was designed to be declarative to make it easy for database administrators to access data without worrying about the underlying execution plan. We'll see some examples of SQL a bit later on.

Relational DBMSes are also called SQL databases, since today, the query language and the relational model often go hand-in-hand. These terms will be used interchangeably throughout the talk.

#### The SQL Standard

SQL's been standardized since the late 80s, but at that point it had already been defined by its implementations for about a decade. Even though SQL:2016 added native support for JSON, Postgres and other databases built in their own support with their own syntax beforehand. That's even before considering the procedural extensions built on top.

SQL-92 is the last, truly standard SQL specification that has come out.

For this talk, we'll be focusing on Postgres's flavor of SQL for simplicity's sake, since it's one of the most popular databases among developers.

### NoSQL

All SQL databases use the same query language and data model and can be more-or-less directly compared to each other. In contrast, NoSQL is an incredibly broad label that can refer to any database system that doesn't use SQL as its query language. This generally maps on to the concept of "non-relational" databases, but even this is extremely broad, covering everything from column-oriented databases like Cassandra, to key/value stores like Redis, and document databases like MongoDB and Fauna.

### Our comparison today

All of these NoSQL databases are targeted towards different use cases, but for the sake of time and all of our sanities today, we'll be focusing on MongoDB, a document database, and its query language, MQL. The smaller but still broad category of document database also includes RedisJSON, Couchbase, Fauna, and Amazon DocumentDB, to name a few.

SQL is standard enough that even though I'll be showing showing Postgres queries, they may very well run unmodified on a bunch of other SQL databases.

Document databases don't share any query language, so all the Mongo queries will be pretty specific to Mongo.

## Databases: what are they good for?

Almost every application that uses a database handles four basic operations commonly called the CRUD operations: Create, Retrieve, Update, and Delete. Some apps are "read heavy" while others are "write heavy". The rates at which different operations are performed is one of the main things that can inform the choice of database and data model.

### A data modeling case study

No matter what kind of database you use, you'll have to design a data model for your application. This is a description of the information you're storing in your database, and how different pieces of information are related to each other. Throughout this talk, we'll use the example of a completely hypothetical course review system I'll call the Cornell Course Cart, or C3. Users of this site can perform two actions:

1. View reviews for courses.
2. Add courses to their course cart.

### Relationships among data

In any data model, there are three fundamental types of relationship among different pieces of data:

- One-to-one: A user has one cart.
- One-to-many: A course can have multiple sections, but each section only belongs to one course.
- Many-to-many: A user's cart can contain many courses, and a course can be in many different users' carts.

#### Storing data in SQL

Like its name suggests, relational databases put the relations between tables front-and-center:

- Data stored in **tables**
- Each entry is a **row**, and every attribute is a **column**
- Each table has a _fixed_ set of columns known as its **schema**.
- Data is modeled primarily through the _relationships_ between tables.

#### Storing data in MongoDB

- Data stored in **collections**.
- Each entry is a **document**, and every attribute is a **field**.
- Every document in a collection can have a **flexible schema**.
- Like JSON, documents can contain **nested fields** and **arrays of subobjects**, which are the natural way to express _relationships_.

### Building a relational schema

Since tables are flat and must have a fixed set of columns, complex applications build up their data model from many tables that have relationships between them. With our course review application we described above, here's what our database schema would look like, once it's populated with some rows:

![](/blog/images/sql-schema.png)

If I want to find out which class a student registered for instead of CIS189, I can follow the `section_id` field to the corresponding row on the `SECTIONS` table, and then follow that row's `course_id` to the `COURSES` table, where I'll find the name and number for the course.

A key property you'll notice of this database schema is that there's no duplicated data anywhere. The only place you can find a course's title is in the COURSE table. The only place to find which professors teach a class is in each SECTION table.

This is a sign that the database is properly _normalized_. Eliminating data redundancy makes managing and updating data much easier, since any piece of information only needs to be updated in one row, in one table. compared to a duplicated or denormalized data model. It's the recommended way to store data in a relational database when starting out.

### Buliding a non-relational schema

```typescript
type course {
	department: string;
	course_number: string;
	title: string;
	sections: [{
		section_number: string;
		professor: string;
		reviews: [{
			...
		}]
	}]
}

type user {
	name: string;
	graduation_year: string;
	cart: {
		courses: [{department: string, course_number: string}];
	}
}
```

You can see here that one-to-many relationships, which required foreign keys in the relational schema, are expressed naturally as nested sub-documents within an array.

These documents can be easily serialized as JSON hand handled by any client programming language that has a JSON parser.

What is expressed a bit more awkwardly, though, is the course cart. What would happen if a course changed its number, or title?

The user<->course relationship is a many-to-many relationship, and the way I've decided to represent the cart has resulted in some data duplication. This makes it fast and easy to access both a user's cart and a course's info, but it requires updating the information in two places if it ever changes. This is called denormalization, and it's a natural pattern in non-relational databases.

### Querying a relational DB

```sql
select *
from COURSES
    inner join SECTIONS on SECTIONS.course_id = COURSES.id
    inner join REVIEWS on SECTIONS.review_id = REVIEWS.id
where
	COURSES.department = “CIS”
	and COURSES.course_code = “188”;

```

Here you can see the declarative nature of SQL. Rather than writing for loops where we build up our result set, we declare what fields we want returned to us, and that we will need to join together the COURSE, SECTION and REVIEW tables to retrieve the information we need. `JOIN`s are **the** critical operation in relational databases. They allow us to construct the logical objects that our application requires without having to pull each table into our application and do the equivalent of a join manually there.

Making the JOIN operation as efficient as possible is an extremely large component of SQL query optimization and database theory. Even so, JOINs remain expensive, and when applications reach a certain scale they will begin to denormalize their data in order to reduce the number of JOINs and be able to process more read queries per second.

### Querying MongoDB

```js
db.courses.find({ department: "CIS", course_number: "188" });
```

No JOINs necessary here! This MongoDB query will pull out all documents where the department and course number match.

Mongo's query language lets you query nested documents very easily. This implicit array traversal is super useful for users, but it's something that's become pretty thorny for us on the optimization team.

### Writing to a relational DB

```sql
begin;
update SECTIONS set professor = "Campbell Phalen"
	inner join COURSES on SECTIONS.course_id = COURSES.id
	where COURSES.department = "CIS"
		and COURSES.course_number = "188"
		and SECTIONS.professor = "Armaan Tobaccowalla"

update SECTIONS set professor = "Rohan Gupta"
	inner join COURSES on SECTIONS.course_id = COURSES.id
	where COURSES.department = "CIS"
		and COURSES.course_number = "188"
		and SECTIONS.professor = "Peyton Walters"
commit;
```

Notice here that our two updates are wrapped in a `begin...commit` block. This is a **transaction**. Transactions bundle multiple operations into a single all-or-nothing operation. If something goes wrong during either of these updates, the database will be rolled back to the state before either of these operations. After a transaction is committed or aborted, we'll never have a situation where Campbell is teaching two sections and Peyton is teaching another one.

### Updating MongoDB

```js
db.courses.updateOne(
	{department: "CIS", course_code: "188"},
	{$set: {
	 "sections.$[peyton].professor": "Rohan Gupta",
	 "sections.$[armaan].professor": "Campbell Phalen"
	 }},
    {arrayFilters: [
		 {"peyton": {"sections.professor": "Peyton Walters"},
		 {"armaan": {"sections.professor": "Armaan Tobaccowalla"},
	}]})
```

You can see that there wasn't any need for a transaction here, since all of the sections live in the same document, and MongoDB's storage engine enforces atomicity of operations on single documents.

### Database normalization

The big benefit of normalization in SQL databases is that whenever a piece of information needs to be updated, it only needs to be updated in a single place. When information is changing frequently, this is a really helpful property. For those who subscribe to the "Don't Repeat Yourself" principle, normalization feels really natural. But it's not the whole story.

<br>

| SQL Databases                                                                                         | MongoDB                                                                                  |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Data is most naturally stored in normalized, de-duplicated flat tables, regardless of access pattern. | Data is most naturally stored how objects are accessed by applications.                  |
| Rely more heavily on joins to **_read normalized data_** into a usable form by the application.       | Can de-emphasize joins and **_write denormalized data_** how it will eventually be used. |

<br>

### Handling semi-structured data in Postgres

I've simplified the data model in the example pretty significantly. One glaring example is how we store review data on each section as a single text field. In a more realistic model, we want to have reviewers be able to give quantitative feedback across a wide array of dimensions. How can we model that? Well, we can make a REVIEW table with a bunch of fields for each dimension, along with a text field.

```text
Instructor quality
Course quality
Communication ability
Course stimulated interest
Instructor was accessible
Readings had value
Amount learned
Would recommend to someone in the major
Would recommend to a non-major
```

But not all these fields are mandatory. Lots of them are optional. And now our table has something like 30 fields on it... computers can handle that fine, but what if there were 500 fields, or 10,000? _Relational databases have trouble handling sparse, semi-structured data_. Now what can we do about this?

Since 2014, Postgres has supported the `jsonb` column type, that can hold unstructured and semi-structured JSON data in regular SQL tables.

### Joins and Transactions in MongoDB

While the document model makes one-to-many and one-to-one relationships trivial to express and work with, most applications will have some many-to-many relationships that will either require full denormalization or some cross-collection operations to work well.

A big misconception about MongoDB is that it can't support JOINs or transactions like a relational database can. It in fact can support these use cases and operations. We can see an example with our user's cart. In order to show all the details of courses in the cart, we'll need to join the user and course collections:

```js
db.users.aggregate([
	{$match: {name: "Campbell Phalen"}},
	{$unwind: "$cart.courses"},
	{$lookup: {
		from "courses",
		let: {department: "$department", course_number: "$course_number"},
		pipeline: [
			{$match: {department: "$$department", course_number: "$$course_number"}}
		]
	}}
])
```

It all comes back to data access patterns and how much you're comfortable going against the grain of your chosen technology. If we can determine that people will be viewing individual sections more often than they'll be viewing their carts, this more expensive operation for getting a user's cart would be acceptable. If people are viewing their carts a lot, we could consider denormalizing the data, and updating course data in the course collection and in each user's cart.

### So, what's _better_?

Each of these systems has its benefits and downsides depending on the kind of application you're building and the kind of data you're looking to handle.

These days, all the dop databases are multi-model. Postgres can store documents in jsonb fields and act like a document database. MongoDB can join multiple collections together in a query and act like a relational database. It's a question of going with the grain and against the grain in whatever system you choose.
