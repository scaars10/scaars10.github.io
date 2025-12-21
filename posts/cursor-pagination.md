# Solving the Deep Paging Problem: Implementing Generic Cursor Pagination at Scale

---

I faced a classic problem while working on a data query platform : building support for arbitrary queries with efficient pagination across diverse datasets.

Given the performance implications of deep paging and the need for scalable data access, we had to move beyond standard approaches. Here is how we implemented a generic cursor-based solution that scales.

---

## The Problem with Skip-Limit

Skip-limit pagination is MongoDB's most straightforward approach:

```js
db.collection.find().skip(N).limit(M)
```

However, internally MongoDB fetches **`N + M` documents and discards the first `N`**. For example:

* Page size = 1000
* Current page = 50
* Requested documents = 51,000
* Dropped documents = 50,000

As pagination moves deeper, performance degrades. In our case, users needed to paginate through hundreds of thousands of records with arbitrary sort orders. Skip-limit simply couldn't scale—page 100 would take orders of magnitude longer than page 1.

## Cursor-Based Pagination

### The Core Concept

Instead of skipping documents, cursor-based pagination continues from the last seen document using filter conditions. The key insight: **convert pagination into a range query**.

### Basic Implementation (No User Sort Key)

When no sort key is specified, we use `_id` as the natural ordering:

```java
// First page
FindIterable<Document> firstPage = collection.find()
    .sort(Sorts.ascending("_id"))
    .limit(pageSize);

// Subsequent pages - continue from last _id
Bson queryForNextPage = Filters.gt("_id", lastId);

FindIterable<Document> nextPage = collection.find(queryForNextPage)
    .sort(Sorts.ascending("_id"))
    .limit(pageSize);
```

---

## Handling Multiple Sort Keys

The complexity increases significantly when users specify custom sort orders. Consider sorting by:

* `unit_price` (ascending)
* `rating` (ascending)

### The Algorithm

To implement cursor pagination with multiple sort keys, we need to:

1. **Always append `_id` as a tiebreaker** to ensure deterministic ordering
2. **Build a compound OR condition** that respects the sort hierarchy

Here's the key logic from our implementation:

```java
public static Bson getAdditionalPaginationCondition(
        Map<String, Object> lastDocMap,
        List<SortCondition> sortConditions) {

    List<SortCondition> sorts = CollectionUtils.isEmpty(sortConditions) 
        ? new ArrayList<>() 
        : new ArrayList<>(sortConditions);

    // Always append _id to handle ties
    sorts.add(new SortCondition(MONGO_ID, SortOrder.ASC));

    List<Bson> orConditions = new ArrayList<>();
    
    // For each sort key, build a condition that handles the hierarchy
    for (int currentSortIdx = 0; currentSortIdx < sorts.size(); currentSortIdx++) {
        List<Bson> subAndConditions = new ArrayList<>();
        
        // All higher-priority sorts must be equal
        for (int higherPrioritySortIdx = 0; higherPrioritySortIdx < currentSortIdx; higherPrioritySortIdx++) {
            subAndConditions.add(getEqPaginationCondition(
                sorts.get(higherPrioritySortIdx), lastDocMap));
        }
        
        // Current sort key must be greater/less than last value
        subAndConditions.add(getRangePaginationCondition(
            sorts.get(currentSortIdx), lastDocMap));

        Bson subCondition = (subAndConditions.size() == 1) 
            ? subAndConditions.get(0) 
            : Filters.and(subAndConditions);
            
        orConditions.add(subCondition);
    }
    
    return (orConditions.size() == 1) 
        ? orConditions.get(0) 
        : Filters.or(orConditions);
}
```

### Breaking Down the Logic

For sorting by `unit_price` (ASC), `rating` (ASC), `_id` (ASC), the generated filter is:

```java
Filters.or(
    // Continue from next unit_price
    Filters.gt("unit_price", lastUnitPrice),
    
    // OR same unit_price but higher rating
    Filters.and(
        Filters.eq("unit_price", lastUnitPrice),
        Filters.gt("rating", lastRating)
    ),
    
    // OR same unit_price and rating but higher _id
    Filters.and(
        Filters.eq("unit_price", lastUnitPrice),
        Filters.eq("rating", lastRating),
        Filters.gt("_id", lastId)
    )
);
```

This pattern scales to any number of sort keys automatically.

---

## Implementation Details

### Query Building

Our query builder handles the complete pagination flow:

```java
protected static MongoQuery buildSearchDatasetWithPaginationQuery(
        List<String> selection,
        List<Condition> conditions,
        List<SortCondition> sort,
        Integer limit,
        Map<String, Object> paginationMetadata) {
    
    MongoQuery.MongoQueryBuilder mongoQueryBuilder = MongoQuery.builder();
    
    addLimit(mongoQueryBuilder, limit);
    
    if (ObjectUtils.isNotEmpty(paginationMetadata)) {
        // Combine user filters with pagination conditions
        addConditionForPagination(mongoQueryBuilder, conditions, sort, paginationMetadata);
    } else {
        addConditions(mongoQueryBuilder, conditions);
    }
    
    // Always include _id in sort
    addSortForPagination(mongoQueryBuilder, sort);
    
    // Include sort keys in projection for pagination metadata
    addSelectionForPagination(mongoQueryBuilder, selection, sort);
    
    return mongoQueryBuilder.build();
}
```

### Pagination Metadata Management

The response includes metadata for the next request:

```java
public static DataStoreResponse parseSearchDatasetWithPaginationResponse(
        MongoCursor<Document> cursor,
        List<SortCondition> sorts,
        List<String> selection,
        Map<String, Object> paginationMetadata,
        int limit) {

    List<Map<String, Object>> documentResponseList = new ArrayList<>();
    Document lastDoc = null;
    
    while (cursor.hasNext()) {
        Document document = cursor.next();
        
        // Track last document for pagination metadata
        if (!cursor.hasNext()) {
            lastDoc = new Document(document);
        }
        
        // Filter to user-requested fields
        if (CollectionUtils.isNotEmpty(selectionSet)) {
            document.keySet().removeIf(key -> !selectionSet.contains(key));
        }
        
        documentResponseList.add(document);
    }

    // Build metadata for next request
    if (ObjectUtils.isNotEmpty(lastDoc)) {
        Map<String, Object> paginationOffsetMap = getPaginationMetadataForMongo(lastDoc, sorts);
        updatedPaginationMetadata.put(PaginationOffsets.OFFSET_VALUE.getValue(), paginationOffsetMap);
    }
    
    updatedPaginationMetadata.put(PaginationConstants.HAS_MORE_DATA, 
        documentResponseList.size() >= limit);
    
    return dataStoreResponse;
}
```

### Field Projection Strategy

A subtle but important detail: we must include sort keys in the database projection even if the user didn't request them:

```java
protected static void addSelectionForPagination(
        MongoQuery.MongoQueryBuilder mongoQueryBuilder,
        List<String> selection,
        List<SortCondition> sorts) {

    if (!CollectionUtils.isEmpty(selection)) {
        Set<String> fieldsRequired = new HashSet<>(selection);
        
        // Include sort keys and _id for pagination metadata
        if (CollectionUtils.isNotEmpty(sorts)) {
            fieldsRequired.addAll(sorts.stream()
                .map(SortCondition::getSortKey)
                .toList());
        }
        
        mongoQueryBuilder.projection(
            Projections.fields(Projections.include(fieldsRequired.stream().toList())));
    }
}
```

These fields are stripped from the response but stored in pagination metadata for the next request.

---

## Indexing Strategy

A critical discovery: MongoDB's query planner struggles with complex OR conditions and multiple sort keys. Even with individual field indexes, performance was poor.

We adopted the **ESR (Equality, Sort, Range) Rule** for compound indexes:

1. **Equality**: User filter fields (e.g., `user_id`, `status`)
2. **Sort**: Pagination sort keys (e.g., `unit_price`, `rating`, `_id`)
3. **Range**: Range filters (e.g., `created_at < timestamp`)

This ordering ensures MongoDB can efficiently:
- Use the index for equality filters
- Traverse the index in sort order
- Apply range filters last

**Example compound index** for a query filtering by `status`, sorting by `unit_price` and `rating`:

```js
db.collection.createIndex({ 
    status: 1,        // Equality
    unit_price: 1,    // Sort
    rating: 1,        // Sort
    _id: 1,           // Sort (tiebreaker)
    created_at: 1     // Range
})
```

---

## Consistency and Updates

### Dealing with New Documents

To prevent newly created documents from appearing mid-pagination, we add an upper bound filter:

```java
protected static void addConditionForPagination(
        MongoQuery.MongoQueryBuilder mongoQueryBuilder,
        List<Condition> conditions,
        List<SortCondition> sortConditions,
        Map<String, Object> paginationMetadata) {
    
    Bson filter = getFilterForQueryConditions(mongoQueryBuilder, conditions);
    
    if (ObjectUtils.isNotEmpty(paginationMetadata.get(PaginationOffsets.OFFSET_VALUE.getValue()))) {
        Bson paginationFilter = MongoUtil.getAdditionalPaginationCondition(
            (Map<String, Object>) paginationMetadata.get(PaginationOffsets.OFFSET_VALUE.getValue()),
            sortConditions);
        
        filter = Filters.and(filter, paginationFilter);
    }
    
    mongoQueryBuilder.filter(filter);
}
```

For time-bound consistency, add:

```java
Filters.lte("created_at", initialQueryTimestamp)
```

### Updates: The Open Problem

Handling updates remains complex. When a document's sort key changes, it can:
- Move to a different page
- Cause duplicates or omissions

For our use case (primarily reads with occasional creates), we accepted cursor pagination's consistency model.

---

## Findings

**Before**: Page 10,000 performance could be thousands of times slower.
**After**: Page 10,000 takes the same amount of time as page 1.

By combining dynamic cursor-based filter generation, strategic compound indexes using ESR, and proper projection management, we achieved uniform latency regardless of page depth.

## Takeaways

**Cursor pagination is necessary for scale.** Deep paging fails with skip-limit.

**Always use valid tiebreakers.** Appending `_id` or some other unique field if you have any is non-negotiable for deterministic ordering.

**Indexes matter.** The ESR rule is critical for compound indexes.

**Updates are tricky.** Cursor pagination is consistent for inserts but handling updates that change sort order requires accepting some trade-offs.