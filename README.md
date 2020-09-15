## Query cheatsheet redash.io mongodb for User Metrics

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
5000	  30/09/20
4000	  31/08/20	
3000	  31/07/20	
2000	  30/06/20	
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
As you can see from the structure of our collection at the top, We had embedded array documents which are called "addonSubscriptions". Its additional subscription documents related to subscription_packages, it's a document for users who had many types of the subscription package. And also we have a "subscription." which our main subscription package. so if we want to calculate the revenue, we need to look up to "subscription_packages" collections and performs a left outer join in the same database to filter in documents from the "joined" collection for processing ($lookup). after that we deconstruct ($unwind) an array field from both to do the calculation from both "subscription." and addonSubscriptions".

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
$40000  	  30/04/19 00:00	
$30000	    31/10/18 00:00	
$20000  	  31/01/20 00:00
```

### Total Revenue
To calculate total of all revenue, it's quite similar like what we use when calculating monthly revenue, we can just use $sum and grouping all of together. $addFields use to adds new fields to documents to have total for all revenue. 

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
totalAllRevenue	_id	 total_addons_revenue	 total_base_revenue	
1,000,000            500,000               500,000 
```

### Total Active customers
We simply sum group from total_addons_user and total_base_user and to define active we need to make sure that the expire/end date greater than today. and then we add results both of them in total_active_user
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
            "$addFields": {
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
total_base_user	 total_active_user	 total_addons_user	
500              11,00                      600
```


### Total monthly paid subscriptions, registered and trial canceled 
We use the same way to perform the monthly query and $facet to have separate aggregation, but there is also $setUnion which performs set operation on arrays, treating arrays as sets. Basically, we gonna join the result from $facet and we don't want to have a duplicate date. $setUnion helps us to filters out duplicates in its result to output an array that contains only unique entries.  

```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$facet": {
                "total_signup": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
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
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": "$monthly",
                            "count_registered": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "total_trial_canceled": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
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
                                                "$year": "$subscription.cend"
                                            },
                                            "month": {
                                                "$add": [
                                                    {
                                                        "$month": "$subscription.cend"
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
                            "count_trial_canceled": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "total_subscribed": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": true
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
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": "$monthly",
                            "count_paid_susbscribed": {
                                "$sum": 1
                            }
                        }
                    }
                ]
            }
        },
        {
            "$project": {
                "activity": {
                    "$setUnion": [
                        "$total_signup",
                        "$total_trial_canceled",
                        "$total_subscribed"
                    ]
                }
            }
        },
        {
            "$unwind": "$activity"
        },
        {
            "$group": {
                "_id": "$activity._id",
                "details": {
                    "$push": "$$ROOT"
                }
            }
        },
        {
            "$project": {
                "registered": {
                    "$arrayElemAt": [
                        "$details.activity.count_registered",
                        0
                    ]
                },
                "paid_subscribed": {
                    "$arrayElemAt": [
                        "$details.activity.count_paid_susbscribed",
                        0
                    ]
                },
                "trial_canceled": {
                    "$arrayElemAt": [
                        "$details.activity.count_trial_canceled",
                        0
                    ]
                }
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

Sample Output
```
_id	          trial_canceled	registered	paid_subscribed	
2020-10-31	    30		           200	        40 
2020-09-30 	    40	             100	        10	
...
```

### Trial to Buy Conversion %
### To do

### Customer Acquisition Cost
```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$facet": {
                "last_month": [
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
                            "addonSubscriptions.cstart": {
                                "$gte": {
                                    "$humanTime": "60 days ago 00:00"
                                },
                                "$lte": {
                                    "$humanTime": "30 days ago 00:00"
                                }
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": null,
                            "count_paid_lastmonth": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "this_month": [
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
                            "addonSubscriptions.cstart": {
                                "$gte": {
                                    "$humanTime": "30 days ago 00:00"
                                },
                                "$lte": {
                                    "$humanTime": "today 00:00"
                                }
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": null,
                            "count_paid_this_month": {
                                "$sum": 1
                            }
                        }
                    }
                ]
            }
        },
        {
            "$project": {
                "_id": 1,
                "paid_last_month": {
                    "$arrayElemAt": [
                        "$last_month.count_paid_lastmonth",
                        0
                    ]
                },
                "paid_this_month": {
                    "$arrayElemAt": [
                        "$this_month.count_paid_this_month",
                        0
                    ]
                }
            }
        },
        {
            "$project": {
                "_id": 1,
                "paid_this_month": 1,
                "paid_last_month": 1,
                "cac_last_month": {
                    "$divide": [
                        {{ expenses_last_month }},
                        "$paid_last_month"
                    ]
                },
                "cac_this_month": {
                    "$divide": [
                        {{ expenses_this_month }},
                        "$paid_this_month"
                    ]
                }
            }
        }
    ]
}
```

Example output
```
paid_last_month	  cac_last_month	cac_this_month	paid_this_month	
4207	               3,8	         1,04           2217	
```

### Monthly active users: sign up, addons user, trial subscription

```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$facet": {
                "monthly_signup": [
                    {
                        "$lookup": {
                            "from": "users",
                            "localField": "user",
                            "foreignField": "_id",
                            "as": "users"
                        }
                    },
                    {
                        "$unwind": {
                            "path": "$users",
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
                                                "$year": "$users.joinDate"
                                            },
                                            "month": {
                                                "$add": [
                                                    {
                                                        "$month": "$users.joinDate"
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
                            "monthly_signup": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "monthly_active_trial_subscribe": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
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
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": "$monthly",
                            "monthly_active_trial_subscribe": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "monthly_active_addons_user": [
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
                        "$project": {
                            "monthly": {
                                "$subtract": [
                                    {
                                        "$dateFromParts": {
                                            "year": {
                                                "$year": "$addonSubscriptions.cstart"
                                            },
                                            "month": {
                                                "$add": [
                                                    {
                                                        "$month": "$addonSubscriptions.cstart"
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
                            "monthly_active_addons_user": {
                                "$sum": 1
                            }
                        }
                    }
                ]
            }
        },
        {
            "$project": {
                "activity": {
                    "$setUnion": [
                        "$monthly_signup",
                        "$monthly_active_trial_subscribe",
                        "$monthly_active_addons_user"
                    ]
                }
            }
        },
        {
            "$unwind": "$activity"
        },
        {
            "$group": {
                "_id": "$activity._id",
                "details": {
                    "$push": "$$ROOT"
                }
            }
        },
        {
            "$project": {
                "_id": 1,
                "monthly_signup": {
                    "$arrayElemAt": [
                        "$details.activity.monthly_signup",
                        0
                    ]
                },
                "monthly_active_trial_subscribe": {
                    "$arrayElemAt": [
                        "$details.activity.monthly_active_trial_subscribe",
                        0
                    ]
                },
                "monthly_active_addons_user": {
                    "$arrayElemAt": [
                        "$details.activity.monthly_active_addons_user",
                        0
                    ]
                }
            }
        },
        {
            "$project": {
                "_id": 1,
                "monthly_signup": 1,
                "monthly_active_trial_subscribe": 1,
                "monthly_active_addons_user": 1,
                "monthly_active_users": {
                    "$add": [
                        "$monthly_active_trial_subscribe",
                        "$monthly_active_addons_user"
                    ]
                }
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

Example output

```
monthly_active_trial_subscribe	_id	        monthly_signup	monthly_active_users	monthly_active_addons_user
100	                            2020-09-30	500             1000	                1000
2000	                          2020-08-31	5000	          10000	                11000
1000	                          2020-07-31	6000	          7000	                4000
```

### ARPU: monthly include MRR

```
{
    "collection": "user_subscriptions",
    "aggregate": [
        {
            "$facet": {
                "revenue_from_subscription": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
                            }
                        }
                    },
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
                            "total_price": {
                                "$multiply": [
                                    "$subscription.quantity",
                                    "$base_subscription.baseCharge"
                                ]
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": "$monthly",
                            "total_revenue": {
                                "$sum": "$total_price"
                            }
                        }
                    }
                ],
                "revenue_from_addons": [
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
                        "$lookup": {
                            "from": "subscription_packages",
                            "localField": "addonSubscriptions.package",
                            "foreignField": "_id",
                            "as": "addons_subscription"
                        }
                    },
                    {
                        "$project": {
                            "monthly": {
                                "$subtract": [
                                    {
                                        "$dateFromParts": {
                                            "year": {
                                                "$year": "$addonSubscriptions.cstart"
                                            },
                                            "month": {
                                                "$add": [
                                                    {
                                                        "$month": "$addonSubscriptions.cstart"
                                                    },
                                                    1
                                                ]
                                            }
                                        }
                                    },
                                    86400000
                                ]
                            },
                            "total_price": {
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
                            "_id": "$monthly",
                            "total_revenue": {
                                "$sum": "$total_price"
                            }
                        }
                    }
                ],
                "active_users": [
                    {
                        "$match": {
                            "addonSubscriptions.0": {
                                "$exists": false
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
                            }
                        }
                    },
                    {
                        "$group": {
                            "_id": "$monthly",
                            "total_active": {
                                "$sum": 1
                            }
                        }
                    }
                ],
                "active_addons_users": [
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
                        "$project": {
                            "monthly": {
                                "$subtract": [
                                    {
                                        "$dateFromParts": {
                                            "year": {
                                                "$year": "$addonSubscriptions.cstart"
                                            },
                                            "month": {
                                                "$add": [
                                                    {
                                                        "$month": "$addonSubscriptions.cstart"
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
                            "total_active": {
                                "$sum": 1
                            }
                        }
                    }
                ]
            }
        },
        {
            "$project": {
                "activity": {
                    "$setUnion": [
                        "$revenue_from_subscription",
                        "$revenue_from_addons",
                        "$active_users",
                        "$active_addons_users"
                    ]
                }
            }
        },
        {
            "$unwind": "$activity"
        },
        {
            "$group": {
                "_id": "$activity._id",
                "details": {
                    "$push": "$$ROOT"
                }
            }
        },
        {
            "$project": {
                "mrr": {
                    "$add": [
                        {
                            "$arrayElemAt": [
                                "$details.activity.total_revenue",
                                0
                            ]
                        },
                        {
                            "$arrayElemAt": [
                                "$details.activity.total_revenue",
                                0
                            ]
                        }
                    ]
                },
                "accounts": {
                    "$add": [
                        {
                            "$arrayElemAt": [
                                "$details.activity.total_active",
                                0
                            ]
                        },
                        {
                            "$arrayElemAt": [
                                "$details.activity.total_active",
                                0
                            ]
                        }
                    ]
                }
            }
        },
        {
            "$project": {
                "_id": 1,
                "mrr": 1,
                "accounts": 1,
                "arpu": {
                    "$divide": [
                        "$mrr",
                        "$accounts"
                    ]
                }
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
## To do
