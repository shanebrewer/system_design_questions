# Order API

Authentication required

POST /v1/Order?ticker={:ticker}&order_direction{:order_direction}&order_type{:order_type}&price{:price}

## Parameters
    OrderDirection: Buy, Sell
    OrderType: Market, Limit, Stop, StopLimit
    (Optional) Price: float

## Responses

### Body

    OrderID: The ID of the order
    creationTime: 

### Codes

    201: Created
    4XX: Client error, Unauthorized, Insufficent Funds
    5XX: Server error

 