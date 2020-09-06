## Query cheatseat redash.io

First let's assume we have this data structure

```
subscription_packages >
  _id
  baseCharge
  category
  enabled
  name
  platforms

user_subscriptions >
  addonSubscriptions >
    category
    package
    enabled
    cstart
    quantity
    cend
    unsubscribed

  _id	
  subscription.package
  subscription.enabled
  subscription.cstart
  subscription.unsubscribed
  subscription.cend
  subscription.quantity

streams >
  _id
  creationTime
  enabled
  key
  lastUpdated
  name
  platforms
  user

users >
  _id
  email
  enabled
  joinDate
  name


```
#### Sum total user
To Sum the total user is quite simple, aggregates the data, and calculates the sum of the data. Keep the id 'null' so it will perform a count, not the list of id. 
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

#### Sum total user with $match filter
We can also use a $match to filters the documents to pass only the documents that match the specified condition(s) to the next pipeline stage.
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

#### Sum total user with multiple $match filter
Not only one filter, you could also perform multiple filter by 
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

#### Monthly signup
To show monthly sign up we need to aggregate and constructs a date to get the last day of the current month. 
```
{
    "collection": "users",
    "aggregate": [
        {
            "$project": {
                "monthly": {
                    "$subtract": [
                        {
                            "$dateFromParts": {
                                "year": {
                                    "$year": "$joinDate"
                                },
                                "month": {
                                    "$add": [
                                        {
                                            "$month": "$joinDate"
                                        },
                                        1
                                    ]
                                }
                            }
                        },
                        86400000
                    ]
                }
            }
        },
        {
            "$group": {
                "_id": "$monthly",
                "count": { "$sum": 1 }
            }
        },
        {
            "$sort": [
                {
                    "name": "_id",
                    "direction": -1
                }
            ]
        }
    ]
}
```
Output example
```
count   _id 
5000	30/09/20
4000	31/08/20	
3000	31/07/20	
2000	30/06/20	
```

### Monthly Signup with date range. 
We could also add $match filter to show data from data range 

```
{
    "collection": "users",
    "aggregate": [
        {
            "$match": {
                "addonSubscriptions.0": {
                    "$exists": false
                },
                "subscription.cend": {
                    "$gte": {
                        "$humanTime": "{{ from_date }} 00:00"
                    },
                    "$lte": {
                        "$humanTime": "{{ to_date }} 00:00"
                    }
                }
            }
        },
        {
            "$project": {
                "monthly": {
                    "$subtract": [
                        {
                            "$dateFromParts": {
                                "year": {
                                    "$year": "$joinDate"
                                },
                                "month": {
                                    "$add": [
                                        {
                                            "$month": "$joinDate"
                                        },
                                        1
                                    ]
                                }
                            }
                        },
                        86400000
                    ]
                }
            }
        },
        {
            "$group": {
                "_id": "$monthly",
                "count": {
                    "$sum": 1
                }
            }
        }
    ]
}

```

### Today and Yesterday Signup
We can use $facet to create multi-faceted aggregations which characterize data across multiple dimensions within a single aggregation stage. So first we will aggregate today signup and yesterday signup then combine the data by Adds new fields to documents using $addfield 

```
{
    "collection": "users",
    "aggregate": [
        {
            "$facet": {
                "today_signup": [
                    {
                        "$match": {
                            "$and": [
                                {
                                    "joinDate": {
                                        "$gte": {
                                            "$humanTime": "today 00:00"
                                        }
                                    }
                                }
                            ]
                        }
                    },
                    {
                        "$group": {
                            "_id": null,
                            "count": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "yesterday_signup": [
                    {
                        "$match": {
                            "$and": [
                                {
                                    "joinDate": {
                                        "$lt": {
                                            "$humanTime": "today 00:00"
                                        },
                                        "$gte": {
                                            "$humanTime": "yesterday 00:00"
                                        }
                                    }
                                }
                            ]
                        }
                    },
                    {
                        "$group": {
                            "_id": null,
                            "count": {
                                "$sum": 1
                            }
                        }
                    }
                ]
            }
        },
        {
            "$addFields": {
                "today_signup": {
                    "$arrayElemAt": [
                        "$today_signup.count",
                        0
                    ]
                }
            }
        },
        {
            "$addFields": {
                "yesterday_signup": {
                    "$arrayElemAt": [
                        "$yesterday_signup.count",
                        0
                    ]
                }
            }
        }
    ]
}

```


