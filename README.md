## Redash.io query cheat sheets with Mongodb

##### Calculate total user
Calculate total user is simple, aggregate the data and calculate the sum of the data. Keep the id 'null' so it will erform a count, not the list of id. 
```
{
    "collection": "users",
    "aggregate": [
        {
            "$group": {
                "count": { "$sum": 1 },
                "_id": null
            }
        }
    ]
}
```

##### Calculate total user with filter enable
We can also use $match to filters the documents to pass only the documents that match the specified condition(s) to the next pipeline stage.
```
{
    "collection": "users",
    "aggregate": [
        {
            "$match": {
                "enabled": true
            }
        },
        {
            "$group": {
                "count": {
                    "$sum": 1
                },
                "_id": null
            }
        }
    ]
}
```

##### Calculate total user with multiple filter
Not only one filter, you could also perform multiple filter. 
```
{
    "collection": "users",
    "aggregate": [
        {
            "$match": {
                "enabled": true,
                "joinDate": {
                    "$lt": {
                        "$humanTime": "today 00:00"
                    },
                    "$gte": {
                        "$humanTime": "yesterday 00:00"
                    }
                }
            }
        },
        {
            "$group": {
                "count": {
                    "$sum": 1
                },
                "_id": null
            }
        }
    ]
}

```

