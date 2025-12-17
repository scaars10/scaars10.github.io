# Solving the Deep Paging Problem: Implementing Generic Cursor Pagination at Scale

---

This post showcases my findings while implementing generic cursor pagination for MongoDB. We were building a data product where one of the usecases was to support arbitrary queries with efficient pagination.

Given the performance implications of deep paging and the need for scalable data access across datasets of varying characteristics, we evaluated multiple approaches. This post compares the trade-offs between standard skip-limit and cursor-based strategies, specifically when MongoDB’s default mechanisms fail to scale.

---

## Skip–Limit Pagination

### Mechanism

Skip–limit pagination is MongoDB's most straightforward approach to support pagination. It uses two parameters:

* `skip`: Number of documents to omit from the beginning of the result set
* `limit`: Maximum number of documents to return

```js
db.collection.find().skip(N).limit(M)
```

In this query, MongoDB skips the first `N` documents and returns up to `M` documents from the remaining dataset.

However, internally MongoDB fetches **`N + M` documents and discards the first `N`**. For example:

* Page size = 1000
* Current page = 50
* Requested documents = 51,000
* Dropped documents = 50,000

As pagination moves deeper, performance degrades significantly.

### Advantages

* **Simplicity**: Easy to understand and implement.
* **Quick access to early pages**: Performs well when only the first few pages are accessed.
* **Random access**: Allows jumping directly to a page by adjusting `skip`.

### Disadvantages

* **Inefficiency**: The database scans and discards all skipped documents, which is expensive at scale.
* **Non-sequential access issues**: Frequent inserts, updates, or deletes can lead to inconsistent page content or missed documents.

---

## Cursor-Based Pagination

### Mechanism

Cursor-based pagination relies on a cursor—a pointer to a specific document—to navigate through a collection. Instead of skipping documents, it fetches documents **after** the cursor.

Cursors are typically implemented using a unique and monotonic field such as:

* `_id`
* Timestamp

For our use case, we sort by `_id` and use it as the cursor when no explicit sort key is provided by the user.

### Cursor-Based Pagination (No User Sort Key)

```java
// Initial query for the first page
FindIterable<Document> firstPage = collection.find()
    .sort(Sorts.ascending("_id"))
    .limit(pageSize);

// Query for follow-up pages
Bson queryForNextPage = Filters.gt("_id", lastId);

FindIterable<Document> nextPage = collection.find(queryForNextPage)
    .sort(Sorts.ascending("_id"))
    .limit(pageSize);
```

---

### Cursor-Based Pagination with Multiple Sort Keys

Pagination becomes more complex when users specify multiple sort keys. For example, sorting by:

* `unit_price` (ascending)
* `rating` (ascending)

To ensure deterministic ordering, we extend the sort keys by appending `_id` as a tiebreaker so actual sort order is preserved while also making sure we always have unique cursor values for next pagination query no matter the sort keys:

```java
// Initial query
FindIterable<Document> firstPage = collection.find()
    .sort(Sorts.orderBy(
        Sorts.ascending("unit_price"),
        Sorts.ascending("rating"),
        Sorts.ascending("_id")
    ))
    .limit(pageSize);

// Query for follow-up pages
Bson queryForNextPage = Filters.or(
    Filters.gt("unit_price", lastUnitPrice),
    Filters.and(
        Filters.eq("unit_price", lastUnitPrice),
        Filters.gt("rating", lastRating)
    ),
    Filters.and(
        Filters.eq("unit_price", lastUnitPrice),
        Filters.eq("rating", lastRating),
        Filters.gt("_id", lastId)
    )
);

FindIterable<Document> nextPage = collection.find(queryForNextPage)
    .sort(Sorts.orderBy(
        Sorts.ascending("unit_price"),
        Sorts.ascending("rating"),
        Sorts.ascending("_id")
    ))
    .limit(pageSize);
```

This approach is extensible and can be dynamically constructed based on the number of sort keys.

---

### Limitations of Cursor-Based Pagination

Cursor-based pagination can perform worse than skip–limit since the complexity of query increases with the number of sort keys so the need of optimal compound index creation becomes extremely necessary and to add to that, supporting arbitrary user-defined sort combinations can become cumbersome.

---

### Advantages

* **Efficiency**: Continues directly from the last document without rescanning previous records.
* **Scalability**: Performance remains stable regardless of dataset size.
* **Consistency**: More resilient to dataset mutations compared to skip–limit.

### Disadvantages

* **Implementation complexity**: Requires managing cursor state and dynamic query generation.
* **Indexing requirements**: Cursor fields must be indexed.
* **Compound index**: Requires compound indexes that may not always be feasible.

---

## Issues with Consistency

MongoDB does not guarantee consistency during pagination like PIT when **Create, Update, or Delete (CUD)** operations occur concurrently.

Since datasets on our platform primarily deal with **creates and updates**, we focus on handling these two cases.

---

## Dealing with Additions

Handling newly added documents during pagination can be achieved by adding an upper bound filter on either:

* `created_at` timestamp, or
* autogenerated `_id`

If autogenerated `_id` is used, explicit `_id` creation must be avoided and a unique index should be created instead.

### Example Query

```java
Bson queryForNextPage = Filters.and(
    Filters.lte("created_at", initialQueryTimeStamp),
    Filters.or(
        Filters.gt("unit_price", lastUnitPrice),
        Filters.and(
            Filters.eq("unit_price", lastUnitPrice),
            Filters.gt("rating", lastRating)
        ),
        Filters.and(
            Filters.eq("unit_price", lastUnitPrice),
            Filters.eq("rating", lastRating),
            Filters.gt("_id", lastId)
        )
    )
);
```

This ensures documents added after the initial query do not appear in subsequent pages.

---

## Dealing with Updates

Handling updates is significantly more complex, especially for stream-like datasets. Updates can:

* Change sort key values
* Cause documents to move across pages
* Lead to duplicates or missing records

This remains an open problem and requires additional modeling or application-level guarantees but given the use case of our platform, we could live with the level of consistency provided by cursor-based pagination.

---



## Other Approaches Considered

We evaluated other approaches such as:

* Time-series–specific pagination
* Bucket-based modeling

These approaches were dataset-specific or required specialized modeling. Since pagination support is intended as a **general capability** across datasets with varying characteristics, we narrowed the solution to skip–limit and cursor-based pagination as the most extensible options.


## Results

A significant discovery during implementation was that MongoDB's query planner can be inefficient with complex OR logic or multiple sort keys with filters. Because MongoDB’s B-Tree indexing often struggles to utilize multiple indexes for a single query (our initial approach was to generate single indexes for queryable fields of a dataset and let MongoDB’s query planner handle the optimization like we saw with Elasticsearch), we introduced a **Query Optimization Layer**.

We standardized our index builds using the **ESR (Equality, Sort, Range) Rule**:

* **Equality**: Filter fields first (e.g., user_id).
* **Sort**: Pagination sort keys second (e.g., unit_price, _id).
* **Range**: Any range filters (e.g., created_at < timestamp) last.

**The Outcome**: <br>
By adding Query Optimizers and Cursor Based Pagination, we moved away from unpredictable scan times and consistent performance. The result was uniform latency-page 100 now loads as fast as page 1.
