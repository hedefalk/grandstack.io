---
id: graphql-schema-generation-augmentation
title: GraphQL Schema Generation And Augmentation
sidebar_label: Schema Generation And Augmentation
---


`neo4j-graphql.js` can create an executable GraphQL schema from GraphQL type definitions or augment an existing GraphQL schema, adding

- auto-generated mutations and queries (including resolvers)
- ordering and pagination fields
- filter fields

## Usage

To add these augmentations to the schema use either the [`augmentSchema`](neo4j-graphql-js-api.md#augmentschemaschema-graphqlschema) or [`makeAugmentedSchema`](neo4j-graphql-js-api.md#makeaugmentedschemaoptions-graphqlschema) functions exported from `neo4j-graphql-js`.

**`makeAugmentedSchema`** - _generate executable schema from GraphQL type definitions only_

```javascript
import { makeAugmentedSchema } from "neo4j-graphql-js";

const typeDefs = `
type Movie {
    title: String
    year: Int
    imdbRating: Float
    genres: [Genre] @relation(name: "IN_GENRE", direction: "OUT")
    similar: [Movie] @cypher(
        statement: """MATCH (this)<-[:RATED]-(:User)-[:RATED]->(s:Movie) 
                      WITH s, COUNT(*) AS score 
                      RETURN s ORDER BY score DESC LIMIT {first}""")
}

type Genre {
    name: String
    movies: [Movie] @relation(name: "IN_GENRE", direction: "IN")
}`;

const schema = makeAugmentedSchema({ typeDefs });
```

**`augmentSchema`** - _when you already have a GraphQL schema object_

```javascript
import { augmentSchema } from "neo4j-graphql-js";
import { makeExecutableSchema } from "apollo-server";
import { typeDefs, resolvers } from "./movies-schema";

const schema = makeExecutableSchema({
  typeDefs,
  resolvers
});

const augmentedSchema = augmentSchema(schema);
```

## Generated Queries

Based on the type definitions provided, fields are added to the Query type for each type defined. For example, the following queries are added based on the type definitions above:

```graphql
Movie(
  title: String
  year: Int
  imdbRating: Float
  _id: Int
  first: Int
  offset: Int
  orderBy: _MovieOrdering
): [Movie]
```

```graphql
Genre(
  name: String
  _id: Int
  first: Int
  offset: Int
  orderBy: _GenreOrdering
): [Genre]
```

## Generated Mutations

Create, update, delete, and add relationship mutations are also generated for each type. For example:

**Create**

```graphql
CreateMovie(
  title: String
  year: Int
  imdbRating: Float
): Movie
```

> If an `ID` typed field is specified in the type defintion, but not provided when the create mutation is executed then a random UUID will be generated and stored in the database.

**Update**

```graphql
UpdateMovie(
  title: String!
  year: Int
  imdbRating: Float
): Movie
```

**Delete**

```graphql
DeleteMovie(
  title: String!
): Movie
```

**Add / Remove Relationship**

Input types are used for relationship mutations.

_Add a relationship with no properties:_

```graphql
AddMovieGenres(
  from: _MovieInput!
  to: _GenreInput!
): _AddMovieGenresPayload
```

and return a special payload type specific to the relationship:

```graphql
type _AddMovieGenresPayload {
  from: Movie
  to: Genre
}
```

Relationship types with properties have an additional `data` parameter for specifying relationship properties:

```graphql
AddMovieRatings(
  from: _UserInput!
  to: _MovieInput!
  data: _RatedInput!
): _AddMovieRatingsPayload

type _RatedInput {
  timestamp: Int
  rating: Float
}
```

Remove relationship:

```graphql
RemoveMovieGenres(
  from: _MovieInput!
  to: _GenreInput!
): _RemoveMovieGenresPayload
```

> See [the relationship types](#relationship-types) section for more information, including how to declare these types in the schema and the relationship type query API.

## Ordering

`neo4j-graphql-js` supports ordering results through the use of an `orderBy` parameter. The augment schema process will add `orderBy` to fields as well as appropriate ordering enum types (where values are a combination of each field and `_asc` for ascending order and `_desc` for descending order). For example:

```
enum _MovieOrdering {
  title_asc
  title_desc
  year_asc
  year_desc
  imdbRating_asc
  imdbRating_desc
  _id_asc
  _id_desc
}
```

## Pagination

`neo4j-graphql-js` support pagination through the use of `first` and `offset` parameters. These parameters are added to the appropriate fields as part of the schema augmentation process.

## Filtering

The auto-generated `filter` argument is used to support complex field level filtering in queries.

See [the Complex GraphQL Filtering section](graphql-filtering.md) for details.

## Configuring Schema Augmentation

You may not want to generate Query and Mutation fields for all types included in your type definitions, or you may not want to generate a Mutation type at all. Both `augmentSchema` and `makeAugmentedSchema` can be passed an optional configuration object to specify which types should be included in queries and mutations.

### Disabling Auto-generated Queries and Mutations

By default, both Query and Mutation types are auto-generated from type definitions and will include fields for all types in the schema. An optional `config` object can be passed to disable generating either the Query or Mutation type.

Using `makeAugmentedSchema`, disable generating the Mutation type:

```javascript
import { makeAugmentedSchema } from "neo4j-graphql-js";

const schema = makeAugmentedSchema({
  typeDefs,
  config: {
    query: true, // default
    mutation: false
  }
}
```

Using `augmentSchema`, disable auto-generating mutations:

```javascript
import { augmentSchema } from "neo4j-graphql-js";

const augmentedSchema = augmentSchema(schema, {
  query: true, //default
  mutation: false
});
```

### Excluding Types

To exclude specific types from being included in the generated Query and Mutation types, pass those type names in to the config object under `exclude`. For example:

```javascript
import { makeAugmentedSchema } from "neo4j-graphql-js";

const schema = makeAugmentedSchema({
  typeDefs,
  config: {
    query: {
      exclude: ["MyCustomPayload"]
    },
    mutation: {
      exclude: ["MyCustomPayload"]
    }
  }
});
```

See the API Reference for [`augmentSchema`](neo4j-graphql-js-api.md#augmentschemaschema-graphqlschema) and [`makeAugmentedSchema`](neo4j-graphql-js-api.md#makeaugmentedschemaoptions-graphqlschema) for more information.