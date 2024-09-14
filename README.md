# About

Price update app with a validation workflow and a notification system, made in Power Apps.<br>
This app enables Sales people to send price update requests, which can then be reviewed by a BU Manager through the request page.

[demo video]


# Context

Pricing is a crucial part of a company strategy when trying to optimize sales performance.
While price fluctuates based on several conditions such as market trends, seasonality, deals with suppliers, it is essential to keep track of price updates for informed decision-making.
In this regard, a pricing tool can benefit Sales teams by providing two elements:
- a status-based workflow that every price request has to go through, in order to facilitate coordination
- a complete visibility of price and request history with date data, in order to facilitate decision-making


# Tools

- [Power Apps](https://www.microsoft.com/en-us/power-platform/products/power-apps)
- [SharePoint (SP)](https://www.microsoft.com/en-us/microsoft-365/sharepoint/collaboration)
- [Power Automate](https://www.microsoft.com/en-us/power-platform/products/power-automate)
- [Teams](https://www.microsoft.com/en-us/microsoft-teams/group-chat-software)


# Diagram

This is a functional overview of the app.<br>
Users only interact with Power App. The SP lists are updated based on request creations and reviews.

![Test](/screenshots/diagram.png)


# Requirements

This is the requirements taken into account for the app development.

|Id | Requirement | Description|
|---|-------------|------------
|1 | Display products with their current price | The product catalog must display all the products with their current price|
|2 | Display product price history | Users must have access to product price history for easier decision-making ||
|3 | Display the requests history | Users must have access to the entire requests history for easier update tracking|
|4 | Review form product info | The request review form must display both the product id and product name|
|5 | Request filters | Users must be able to find specific requests more easily, based on product name/ID, status, their own requests |
|6 | Block request creation on products with a pending request | A product with an already pending request cannot have a new request created for it. Once the pending |request is accepted or rejected/canceled, the user can make a new request for the product|
|6 | Pending request alert in the product catalog | An alert must be displayed on the product card to indicate the product already has a pending request.|
|7 | Pending alert tooltip | When a product already has a request pending, the user can hover the alert message to display a tooltip with the date and time at which the request was created.|
|8 | Ability to cancel requests | Users must be able to cancel their own requests while they're still in "Pending" status|
|9 | Refresh data | Users must be able to refresh the product catalog/request page items to make sure they see the most recent data|
|10 | Item counts | Product catalog and requests history must display the total number of products/requests, taking into account filters|
|11 | No items message | Product catalog and requests history must display a message if no product matches the applied filters|
|12 | Searching product/request by product name or id | Users must be able to search for products/requests by the product name or product ID|
|13 | Request history sorted by most recent | More convenient UX-wise|
|14 | Review comment | BU Manager must be able to fill a review comment when rejecting a request, in order to explain why the request was rejected|
|15 | Display success/error messages for request sending and request reviewing | Messages must be directly displayed to inform users their action has succeeded or failed|


# Data modeling

The app relies on three SP lists:
- <b>products</b>, storing the products and their data which are updated by requests
- <b>requests</b>, storing the entire requests history, with every request and its data
- <b>employees</b>, storing the employees information, used to check for role

## Products data dictionary

|Column name | Data type | Description | Example|
|------------|-----------|-------------|--------|
|title | string | Product name | Product 1|
|id | int | The unique and durable identifier of the product | 8|
|price | decimal | The product selling price | 25.99|
|start_date | datetime | The datetime at which the record was created. Marks the point in time when the record started being the "Current" version of the product. |2024-25-10 3:15|
|start_date_notime | date | Same as above, without the time for the linechart x axis | 2024-25-10|
|end_date | datetime | The datetime at which the record was set to status "Expired". Marks the point in time when the record stopped being the "Current" version. The value is "9999-31-12" as long as the status is "Current" | 2024-25-10 3:15|
|status | string | Indicates if the record is the active product version ("Current") or a previous, inactive product version ("Expired") | Current|

[screenshot]

Example: Product 5 had the price $25.99 from 2023-04-05 to 2023-04-10.


## Requests data dictionary

|Column name | Data type | Description | Example|
|------------|-----------|-------------|--------|
|id | int | The unique and durable identifier of the request | 8|
|product_id | int | The unique and durable identifier of the product | 8|
|old_price | decimal | Current price at the moment the request was created | 25.99|
|new_price | decimal | New price to replace old_price, indicated in the request form | 25.99|
|request_date | datetime | Datetime at which the request was created | 2024-25-10 3:15|
|review_date | datetime | Datetime at which the request was reviewed by the BU Manager or canceled by the request author | 2024-25-10 3:15|
|requester_comment | string | Optional comment added in the request form to provide info to the BU Manager | Price decrease for holidays|
|reviewer_comment | string | Optional comment added in the review form to provide info to the sales person when the request is rejected | New price is too low|
|status | string | Workflow status to indicate if the request has been reviewed or not. Can be one of the following values: Pending, Accepeted, Rejected, Canceled | Pending|
|employee_id | int | The unique and durable identifier of the employee who created the request | 8|

[screenshot]

Example: In the request 5, the Sales person requested a price change for the product with id 11. The Sales person requested a price increase from $25.99 to $31.99.
The request was accepted by the BU Manager on 2023-04-05.


## Employees data dictionary

|Column name | Data type | Description | Example|
|------------|-----------|-------------|--------|
|id | int | The unique and durable identifier of the employee | 8|
|first_name | string | Employee's first name | ANNA|
|last_name | string | Employee's last name | JOHNS|
|role | string | Role of the employee | manager|
|email | string | Email of the employee | ANNAJOHNS@annajohn41.onmicrosoft.com|

[screenshot]


## SP Lists update

### Requests list

The requests list is modeled as a type 1 SCD (Slowly Changing Dimension) as seen in dimensional modeling.<br>
In the requests list, a new record is created for every request made in the app.<br>
When it is accepted or rejected/canceled, the request record is updated in the following way:
- set review_date to current datetime value
- set review_comment to the comment specified in the review form by the reviewer
- update status from "Pending" to "Accepted"/"Rejected"/"Canceled"


### Products list

In order to preserve and display product price history, the products SP list is modeled as a type 2 SCD.<br>
Consequently, a product will have a record for every accepted request related to it. The active record is identified by the "Current" status, while all other records for the product will have the "Expired" status to indicate they are the previous and inactive versions of the product.<br>
Product price history is retained through columns ``start_date``, ``end_date``, and ``status`` ("Current"/"Expired").<br><br>

When a request is accepted for a product, the following happens in the products SP list:
- locate and update its product record where status is "Current"
  - update status from "Current" to "Expired"
  - update end_date from "9999/31/12" to the current datetime value
- add a record for the product's new version
  - set title and id by retrieving values from the request
  - set price to the new_price value specified in the request form
  - set start_date to current datetime value
  - set start_date_notime to current date value
  - set end_date to 9999/31/12
  - set status to "Current"



# Notification system

The notification system relies on a Power Automate flow listening to events in the requests SP list.

![Test](/screenshots/flow.PNG)

## Functional description

When a request is created or reviewed, a Teams notification is sent automatically through a Power Automate flow.<br>

The flow works as follows:
1) when a request is created by a sales person, the BU manager receives a Teams notification with the request details
2) when a request is canceled by the sales person who created it, the BU manager receives a Teams notification
3) when a request is reviewed by the BU manager, the sales person who created the request receives a Teams notification with the review details


## Technical description

The flow is triggered by creation and update events in the requests SP list.<br>
Since we want to trigger different notifications for the creation and update events, the flow is split in 2 paths, the creation path and the update path.<br>
The split is based on the "Version Number" property of the event body:<br>

`` "{VersionNumber}": "1.0" ``

- When a new record is created in a SP list, its version number is 1
- A record's version number increases incrementally each time it is updated: since the request review updates the record, it increases its version number from 1 to 2<br>

Consequently:
- a request creation event has a version number of 1
- a request review event has a version number of 2 


[screenshot]


# Code

You will find the code for the main components below, illustrating the business logic of the app.


## Product Catalog

The product catalog lets users browse by scrolling or searching a product name/ID.<br>
Products are displayed with their current price.<br><br>

Component type: Vertical gallery<br>
Property: Items<br>

```
If(
    IsBlank(searchBar_product.Input),

    SortByColumns(
        Filter(
            product_data, 
            status = "Current"
        ), 
        "Title", 
        SortOrder.Ascending),
        
    SortByColumns(
        Filter(
            product_data, 
            status = "Current" && 
            Or(
                searchBar_product.Input in name, 
                Text(id) = searchBar_product.Input
            )
        ), 
        "Title", 
        SortOrder.Ascending)
)

```
[onSelect]



## Requests history

Requests are sorted by most recent, and filtered when filters are used.<br><br>

Component type: Vertical gallery<br>
Property: Items<br>

```
Sort(
    Filter(
        requests,

        // If the searchbar is not empty, look for the product that has a matching name or ID
        If(
            !IsBlank(searchBar_request.Input), 
            Or(
                product_id = LookUp(product_data, name = searchBar_request.Input, id), 
                Text(product_id) = searchBar_request.Input
                ), 
                true
        ) &&

        If(
            filterDD_Status.SelectedValue <> "All",
            status = filterDD_Status.SelectedValue, 
            true
        ) &&

        If(
            filterDD_Employee.SelectedValue = "My requests",
            employee = User().FullName, 
            true
        )
    ),
    request_date,
    SortOrder.Descending
)
```

## Request creation

After the request form is filled, clicking the "Send request" button adds the request as a new record in the requests SP list.<br><br>

Component type: Button<br>
Property: OnSelect<br>

```
Refresh(requests); // Refresh data source for the following check

// Check if a request for the product already exists
If(
    IsEmpty(
        Filter(
            requests,
            product_id = CurrentProductId &&
            status = "Pending"
        )
    ) = true &&

    // New_price value must be a number without text
    IsMatch(InputNewPrice.Value, "\d+(\.\d+)?"),
    Patch(
        requests,
        Defaults(requests),
        {id: Last(requests).id + 1, product_id: CurrentProductId, old_price: CurrentProductPrice, new_price: Int(InputNewPrice.Value), request_date: Now(), status: "Pending", requester_comment: FormComment.Value, employee: User().FullName}
    );
    Notify("Request successfully created", NotificationType.Success, 4000);
    Set(showElement, Blank()),

    IsEmpty(
        Filter(
            requests,
            product_id = CurrentProductId &&
            status = "Pending"
        )
    ) = true &&
    ! IsMatch(InputNewPrice.Value, "\d+(\.\d+)?"),
    Notify("New price must be a number without text", NotificationType.Error, 4000)
);
```


## Request review

The "Accept" and "Reject" actions have different codes to update the lists data accordingly.<br>

Component type: Button<br>
Property: OnSelect<br>

### Accept

Error handling is done with the ```IfError``` function by wrapping the 3 Patch/Notify pairs: if a ```Patch``` fails, the execution stops.<br>

[See documentation](https://learn.microsoft.com/en-us/power-platform/power-fx/reference/function-iferror)

```
IfError(

    // Update request
    Patch(
        requests,
        First(Filter(requests, product_id = CurrentRequestProductId && status = "Pending")),
        {review_date: Now(), status: "Accepted"}
    ),
    Notify("Couldn't update the request from ""Pending"" to ""Accepted""", NotificationType.Error),

    // Update product with the "Current" status
    Patch(
        product_data,
        First(Filter(product_data, id = Int(CurrentRequestProductId) && status = "Current")),
        {end_date: Now(), status: "Expired"}
    ),
    Notify("Couldn't update the product data from ""Current"" to ""Expired""", NotificationType.Error),

    // Add the product new record
    Patch(
        product_data,
        Defaults(product_data),
        {name: LookUp(product_data, id = CurrentRequestProductId, name), id: Int(CurrentRequestProductId), price: CurrentRequestNewPrice, start_date: Now(), end_date: Date(9999, 12, 31), status: "Current"}
    ),
    Notify("Couldn't add the new product data", NotificationType.Error)
);
Notify("Request was successfully accepted", NotificationType.Success); // Display success message only if all the Patches pass

Set(showElement, Blank()) // Closes the form window
```


### Reject

When the request status is "Pending", the "Reject" action has a different outcome based on the user role:
1. if the user is the request author, updates the status to "Canceled"
2. if the user is the BU Manager, updates the status to "Rejected"

Checking user role is done by using the CurrentUserRole global variable set in the ``OnStart`` property of the App

Set(CurrentUserRole, LookUp(employees, email = User().Email).role); 

```
IfError(
    If(
        CurrentUserRole = "manager", // Checks if the user is the BU Manager
        Patch(
            requests,
            First(Filter(requests, product_id = CurrentRequestProductId && status = "Pending")),
            {review_date: Now(), status: "Rejected", reviewer_comment: reviewCommentInput.Value}
        );
        Notify("Request was successfully rejected", NotificationType.Success),

        CurrentUserId = CurrentRequestEmployeeId, // Checks if the user is the request author
        Patch(
            requests,
            First(Filter(requests, product_id = CurrentRequestProductId && status = "Pending")),
            {review_date: Now(), status: "Canceled", reviewer_comment: reviewCommentInput.Value}
        );
        Notify("Request was successfully canceled", NotificationType.Success)
    ),
    Notify("Couldn't update the request", NotificationType.Error)
);

Set(showElement, Blank()) // Closes the form window
```

# Showcase

[screenshots]
