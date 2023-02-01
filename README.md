# What is Fine-Grained Authorization?

Before we understand Fine-Grained Authorization, lets understand the existing Access Control Strategies

Traditionally, we had
1. Role-based Access Control
2. Attribute-based Access Control

**Role-based access control (RBAC), basically, is used to grant access to the resources using group (roles) membership.**  
eg. a ResourceX can specify that all users belonging to RoleX group( say UserA, UserB etc) can access the resource.  
If we need to remove UserA's access, we need to remove him from the RoleX

Obviously, the advantage is simplification of the definition of access, but as we scale out the groups and users, with hierarchies and nesting, it becomes complex very quickly.

**Attribute-based access control (ABAC) introduces access rules based on various factors, eg. the requester identity, and the resource attributes, and even contextual elements like request origin (eg request-time, device, location etc)**
eg. the ResourceX can be deleted only if the request originates from the resource creator,   
also the ResourceX can be accessed only if the request originates from an Android device and so forth

As can be seen from the example above, the implementation is complex for ABAC since its very granular, while revoking or adding permissions is much easier as we need to add/remove attributes 


## When to use RBAC or ABAC?
Even though ABAC is widely considered an evolved form of RBAC, it’s not always the right choice. Depending on your company’s size, budget, and security needs, you may choose one over the other.

Choose ABAC if you:
* Have the time, resources, and budget for a proper ABAC implementation.
* Are in a large organization, which is constantly growing. ABAC enables scalability.
* Have a workforce that is geographically distributed. ABAC can help you add attributes based on location and time-zone.
* Want as granular and flexible an access control policy as possible.
* Want to future-proof your access control policy. The world is evolving, and RBAC is slowly becoming a dated approach. ABAC gives you more control and flexibility over your security controls.

Choose RBAC if you:
* Are in a small-to-medium sized organization.
* Have well-defined groups within your organization, and applying wide, role-based policies makes sense.
* Have limited time, resources, and/or budget to implement an access control policy.
* Don’t have too many external contributors and don’t expect to onboard a lot of new people.

# Fine-Grained Authorization

Fine-Grained Authorization as implemented by AWS IAM, brings the best of both worlds, by making use of a policy language on all resources as well as services

The Policy Language used by AWS IAM, uses four objects
1. The **principal** is the entity taking the action, which can either be a user or another service
2. The **action** is the operation being performed, for which permission must be granted. 
3. The **resource** is the target of the call.
4. The **condition** limits when or where the principal can make the action on the resource.

Using the policy language, we can enable UserA(the principal) to delete(the action) ResourceX (the resource) only when user is logged with MFA (the condition)

Let's Consider some Examples below. I am choosing DynamoDB as the following examples showcase Find-Grained Authorization on Table as well as its Attributes

1. Allow read only Access to a DynamoDB table, and also, limiting access to items which have "BC" as their partition key value  
```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadOnlyAccessToUserItems",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:BatchGetItem",
                "dynamodb:Query"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-west-2:123456789012:table/GameScores"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "dynamodb:LeadingKeys": "BC"
                }
            }
    ]
}
```
2. Grant permissions that limit access to specific attributes in a table
```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LimitAccessToSpecificAttributes",
            "Effect": "Allow",
            "Action": [
                "dynamodb:UpdateItem",
                "dynamodb:GetItem",
                "dynamodb:Query",
                "dynamodb:BatchGetItem",
                "dynamodb:Scan"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-west-2:123456789012:table/GameScores"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "dynamodb:Attributes": [
                        "UserId",
                        "TopScore"
                    ]
                },
                "StringEqualsIfExists": {
                    "dynamodb:Select": "SPECIFIC_ATTRIBUTES",
                    "dynamodb:ReturnValues": [
                        "NONE",
                        "UPDATED_OLD",
                        "UPDATED_NEW"
                    ]
                }
            }
        }
    ]
}
```
3. Grant permissions to prevent updates on certain attributes
```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PreventUpdatesOnCertainAttributes",
            "Effect": "Allow",
            "Action": [
                "dynamodb:UpdateItem"
            ],
            "Resource": "arn:aws:dynamodb:us-west-2:123456789012:table/GameScores",
            "Condition": {
                "ForAllValues:StringNotLike": {
                    "dynamodb:Attributes": [
                        "FreeGamesAvailable",
                        "BossLevelUnlocked"
                    ]
                },
                "StringEquals": {
                    "dynamodb:ReturnValues": [
                        "NONE",
                        "UPDATED_OLD",
                        "UPDATED_NEW"
                    ]
                }
            }
        }
    ]
}
```
4. Grant permissions to query only projected attributes in an index
```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "QueryOnlyProjectedIndexAttributes",
            "Effect": "Allow",
            "Action": [
                "dynamodb:Query"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-west-2:123456789012:table/GameScores/index/TopScoreDateTimeIndex"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "dynamodb:Attributes": [
                        "TopScoreDateTime",
                        "GameTitle",
                        "Wins",
                        "Losses",
                        "Attempts"
                    ]
                },
                "StringEquals": {
                    "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
                }
            }
        }
    ]
}
```