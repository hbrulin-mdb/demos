# Show data in Compass

Now if I look at the sample data I have in my db, I can see that I have 3 collections : 
- accounts : This collection contains account details for users. Each document contains an account id, a limit and the products that a customer has purchased.
- customers : This collection contains customers details including what accounts they hold users. Each document contains username, name, address, birth date, email address, a list of the accounts held, and details on the tiers and related benefits they are entitled to.
- transactions : This collection contains transactions details for users. Each document contains an account id, a count of how many transactions are in this set, the start and end dates for transactions covered by this document, and a list of sub documents. Each sub document represents a single transaction and the related information for that transaction.

## Add a new customer

db.customers.insertOne({
  "username": "jdoe",
  "name": "John Doe",
  "address": "1234 Elm Street\nSpringfield, IL 62704",
  "birthdate": {
    "$date": "1985-07-15T08:45:00.000Z"
  },
  "email": "johndoe@example.com",
  "active": true,
  "accounts": [
    123456,
    789012,
    345678
  ],
  "tier_and_details": {
    "a1b2c3d4e5f67890123456789abcdef0": {
      "tier": "Silver",
      "id": "a1b2c3d4e5f67890123456789abcdef0",
      "active": true,
      "benefits": [
        "priority support",
        "discounted rates"
      ]
    }
  }
});




## Set a validation rule :
```js
const validatorRule = {
      "bsonType": "object",
      "required": ["username", "name", "address", "birthdate", "email", "active", "accounts", "tier_and_details"],
      "properties": {
        "username": {
          "bsonType": "string",
          "description": "must be a string and is required"
        },
        "name": {
          "bsonType": "string",
          "description": "must be a string and is required"
        },
        "address": {
          "bsonType": "string",
          "description": "must be a string and is required"
        },
        "birthdate": {
          "bsonType": "date",
          "description": "must be a date and is required"
        },
        "email": {
          "bsonType": "string",
          "pattern": "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$",
          "description": "must be a valid email format and is required"
        },
        "active": {
          "bsonType": "bool",
          "description": "must be a boolean and is required"
        },
        "accounts": {
          "bsonType": "array",
          "items": {
            "bsonType": "int"
          },
          "description": "must be an array of integers and is required"
        },
        "tier_and_details": {
          "bsonType": "object",
          "additionalProperties": {
            "bsonType": "object",
            "required": ["tier", "id", "active", "benefits"],
            "properties": {
              "tier": {
                "bsonType": "string",
                "enum": ["Bronze", "Silver", "Gold", "Platinum"],
                "description": "must be a valid tier type"
              },
              "id": {
                "bsonType": "string",
                "description": "must be a unique string identifier"
              },
              "active": {
                "bsonType": "bool",
                "description": "must be a boolean"
              },
              "benefits": {
                "bsonType": "array",
                "items": {
                  "bsonType": "string"
                },
                "description": "must be an array of strings"
              }
            }
          },
          "description": "must be an object with tier details"
        }
      }
    }
```

- Ensures required fields exist ("required": [...]).
- Validates data types ("bsonType": "string", "bsonType": "bool", etc.).
- Restricts email format using a regex pattern.
- Enforces the "tier" field to be one of: "Bronze", "Silver", "Gold", or "Platinum".
- Ensures "accounts" is an array of integers.
- Allows "tier_and_details" to contain multiple objects, where each must include "tier", "id", "active", and "benefits".

const schema_validation = { $jsonSchema: validatorRule };

db.runCommand({
    collMod: "customers",
    validator: schema_validation,
    validationLevel: "strict",
    validationAction: "error",
});

mongodb offers validation level options to control the enforcement : 
- strict : applies rules to all inserted and updated documents
- moderate : applies rules to new and existing documents which have been validated 

also two options to define what happens when validation fails : 
- error : reject the update. default
- warn : operation complete but record in log



## Try and add a customer without respecting validation rules : 
db.customers.insertOne({
  "username": "jdoe",
  "name": "John Doe",
  "address": "1234 Elm Street\nSpringfield, IL 62704",
  "birthdate": {
    "$date": "1985-07-15T08:45:00.000Z"
  },
  "email": "johndoe@example.com",
  "active": true,
  "accounts": [
    123456,
    789012,
    345678
  ],
  "tier_and_details": {
    "a1b2c3d4e5f67890123456789abcdef0": {
      "tier": "Apple",
      "id": "a1b2c3d4e5f67890123456789abcdef0",
      "active": true,
      "benefits": [
        "priority support",
        "discounted rates"
      ]
    }
  }
});


## Add a new transaction

db.transactions.updateOne(
  { 
    account_id: 443178, 
    bucket_start_date: { "$lte": ISODate("1969-02-04T00:00:00.000+00:00") }, 
    bucket_end_date: { "$gte": ISODate("2017-01-03T00:00:00.000+00:00") }
  },
  { 
    "$push": { 
      "transactions": {
        "date": ISODate("2023-01-01T00:00:00Z"),
        "amount": 5000,
        "transaction_code": "buy",
        "symbol": "tsla",
        "price": "120.50",
        "total": "602500.00"
      }
    },
    "$inc": { "transaction_count": 1 }
  }
);


# Show some aggregations in Compass

- what is account 443178's largest single transaction?
- most frequent transaction categories

(- monthly spending trend
- total transactions and average transaction value
- total balance per customer across all accounts)

SHOW EXPORT TO CODE

and we could also store that as a view so that we precompute the most frequent queries and can get them without rerunning the aggregation pipeline.  


# TimeSeries

Now as we've seen with our transactions collection, we have been bucketing our transactions with time buckets. In order to avoid our index to grow out of bounds, because these can quickly become millions of data points.

What we can do is for these transactions to be stored as a timeseries. TS collections are optimized for append only patterns. Data automatically clustered by time. bucket pattern improves efficiency. 


```sh
### Data Load

db.createCollection("transactionts", {
  "timeseries": {
    "timeField": "date",     // Field representing the timestamp of each transaction
    "metaField": "account_id", // Optional: Metadata field (e.g., account_id)
    "granularity": "seconds"  // Can be "seconds", "minutes", or "hours"
  }
})

db.transactionts.insertMany([
  {
    "date": ISODate("2024-02-28T14:30:00Z"),
    "account_id": 371138,
    "amount": 500.00,
    "category": "Shopping",
    "description": "Amazon Purchase"
  },
  {
    "date": ISODate("2024-02-28T15:00:00Z"),
    "account_id": 324287,
    "amount": 1200.50,
    "category": "Rent",
    "description": "Monthly Rent Payment"
  },
  {
    "date": ISODate("2024-02-28T16:15:00Z"),
    "account_id": 276528,
    "amount": 60.75,
    "category": "Groceries",
    "description": "Supermarket Purchase"
  }
])

# Get Total Transactions per day

db.transactionts.aggregate([
  {
    "$group": {
      "_id": { "$dateToString": { "format": "%Y-%m-%d", "date": "$date" } },
      "total_spent": { "$sum": "$amount" },
      "transaction_count": { "$sum": 1 }
    }
  },
  { "$sort": { "_id": 1 } }
])

## Get monthly spending for a customer

db.transactionts.aggregate([
  { "$match": { "account_id": 371138 } }, // Filter by customer’s account
  {
    "$group": {
      "_id": { "$dateToString": { "format": "%Y-%m", "date": "$date" } },
      "total_spent": { "$sum": "$amount" },
      "transaction_count": { "$sum": 1 }
    }
  },
  { "$sort": { "_id": 1 } }
])

# a window function

# avg, max, min