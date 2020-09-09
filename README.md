## Query cheatsheet redash.io mongodb for SaaS Metrics

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
We can use $facet to create multi-faceted aggregations that characterize data across multiple dimensions within a single aggregation stage. So first we will aggregate today signup and yesterday signup then combine the data by adds new fields to documents using $addfield 

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
### Total subscriptions by category
The subscription category is located on addonSubscriptions documents. To be able to get the exact documents we need to use $unwind, which is help us to deconstructs an array field from the input documents to output a document for each element. 
```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$unwind": {
                "path": "$addonSubscriptions",
                "preserveNullAndEmptyArrays": false
            }
        },
        {
            "$project": {
                "category": "$addonSubscriptions.category"
            }
        },
        {
            "$group": {
                "_id": "$category",
                "count": {
                    "$sum": 1
                }
            }
        }
    ]
}

```

output

```
1000	categoryA
2000	categoryB
3000	categoryC
4000	categoryD
```
### Monthly Revenue
As you can see from the structure of our collection at the top, We had embedded array documents which are called "addonSubscriptions". Its additional subscription documents related to subscription_packages, it's a document for users who had many types of the subscription package. And also we have a "subscription." which our main subscription package. so if we want to calculate the revenue, we need to look up to "subscription_packages" collections and Performs a left outer join in the same database to filter in documents from the "joined" collection for processing ($lookup). after that we deconstruct ($unwind) an array field from both to do the calculation from both "subscription." and addonSubscriptions".

```

{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$lookup": {
                "from": "subscription_packages",
                "localField": "subscription.package",
                "foreignField": "_id",
                "as": "base_subscription"
            }
        },
        {
            "$unwind": {
                "path": "$base_subscription",
                "preserveNullAndEmptyArrays": false
            }
        },
        {
            "$project": {
                "monthly": {
                    "$subtract": [
                        {
                            "$dateFromParts": {
                                "year": {
                                    "$year": "$subscription.cstart"
                                },
                                "month": {
                                    "$add": [
                                        {
                                            "$month": "$subscription.cstart"
                                        },
                                        1
                                    ]
                                }
                            }
                        },
                        86400000
                    ]
                },
                "addonSubscriptions": 1,
                "total_revenue": {
                    "$multiply": [
                        "$subscription.quantity",
                        "$base_subscription.baseCharge"
                    ]
                }
            }
        },
        {
            "$unwind": {
                "path": "$addonSubscriptions",
                "preserveNullAndEmptyArrays": false
            }
        },
        {
            "$lookup": {
                "from": "subscription_packages",
                "localField": "addonSubscriptions.package",
                "foreignField": "_id",
                "as": "addons_subscription"
            }
        },
        {
            "$project": {
                "monthly": 1,
                "total_revenue": 1,
                "total_revenue_addons": {
                    "$multiply": [
                        {
                            "$arrayElemAt": [
                                "$addons_subscription.baseCharge",
                                0
                            ]
                        },
                        "$addonSubscriptions.quantity"
                    ]
                }
            }
        },
        {
            "$addFields": {
                "totalAllRevenue": {
                    "$add": [
                        "$total_revenue_addons",
                        "$total_revenue"
                    ]
                }
            }
        },
        {
            "$group": {
                "_id": "$monthly",
                "totalRevenue": {
                    "$sum": "$totalAllRevenue"
                }
            }
        }
    ]
}

```

The output will be

```
Total       ID
$50000	    31/05/20 00:00	
$40000  	30/04/19 00:00	
$30000	    31/10/18 00:00	
$20000  	31/01/20 00:00
```

### Total Revenue
To calculate total of all revenue, it's quite similar like what we use when calculating monthly revenue, we can just use $sum and grouping all of together. $addFields use to adds new fields to documents.

```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$lookup": {
                "from": "subscription_packages",
                "localField": "subscription.package",
                "foreignField": "_id",
                "as": "base_subscription"
            }
        },
        {
            "$project": {
                "user": 1,
                "addonSubscriptions": 1,
                "total_revenue": {
                    "$multiply": [
                        {
                            "$arrayElemAt": [
                                "$base_subscription.baseCharge",
                                0
                            ]
                        },
                        "$subscription.quantity"
                    ]
                }
            }
        },
        {
            "$unwind": {
                "path": "$addonSubscriptions",
                "preserveNullAndEmptyArrays": false
            }
        },
        {
            "$lookup": {
                "from": "subscription_packages",
                "localField": "addonSubscriptions.package",
                "foreignField": "_id",
                "as": "addons_subscription"
            }
        },
        {
            "$project": {
                "total_revenue": 1,
                "total_revenue_addons": {
                    "$multiply": [
                        {
                            "$arrayElemAt": [
                                "$addons_subscription.baseCharge",
                                0
                            ]
                        },
                        "$addonSubscriptions.quantity"
                    ]
                }
            }
        },
        {
            "$group": {
                "_id": null,
                "total_addons_revenue": {
                    "$sum": "$total_revenue_addons"
                },
                "total_base_revenue": {
                    "$sum": "$total_revenue"
                }
            }
        },
        {
            "$addFields": {
                "totalAllRevenue": {
                    "$add": [
                        "$total_addons_revenue",
                        "$total_base_revenue"
                    ]
                }
            }
        }
    ]
}

```

Output

```
totalAllRevenue	_id	total_addons_revenue	total_base_revenue	
1,000,000           500,000                     500,000 
```

### Total Active customers

```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$facet": {
                "total_addons_user": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": true
                            }
                        }
                    },
                    {
                        "$unwind": {
                            "path": "$addonSubscriptions",
                            "preserveNullAndEmptyArrays": false
                        }
                    },
                    {
                        "$match": {
                            "addonSubscriptions.cend": {
                                "$gte": {
                                    "$humanTime": "today 00:00"
                                }
                            }
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
                "total_base_user": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
                            },
                            "subscription.cend": {
                                "$gte": {
                                    "$humanTime": "today 00:00"
                                }
                            }
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
                "total_base_user": {
                    "$arrayElemAt": [
                        "$total_base_user.count",
                        0
                    ]
                }
            }
        },
        {
            "$addFields": {
                "total_addons_user": {
                    "$arrayElemAt": [
                        "$total_addons_user.count",
                        0
                    ]
                }
            }
        },
        {
            "$project": {
                "total_base_user": 1,
                "total_addons_user": 1,
                "total_active_user": {
                    "$add": [
                        "$total_base_user",
                        "$total_addons_user"
                    ]
                }
            }
        }
    ]
}
```
Output
```
total_base_user	total_active_user	total_addons_user	
500              11,00                600
```

## To do
