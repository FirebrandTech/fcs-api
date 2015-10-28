# Firebrand Cloud Services API

## Assumptions

This document will help you use the Firebrand Cloud Services API to do the following:

- Authenticate one of your users with Firebrand
- Register an ebook order for your user with Firebrand
- Allow a user to download ebooks they have purchased through Firebrand's library widget

This document assumes you are hosting the library widget on `yourdomain.com` and hitting Firebrand's production API at `cloud.firebrandtech.com`.

## The Authentication API

To authenticate with Firebrand's API, you will need:

- your API Key
- you API Secret
- your Service URL

As an example, we will assume that your API Key is **apikey123** and your secret is **secret789**.  In production, the Service URL is **https://cloud.firebrandtech.com**.  Other URLs might be used during testing.

You will be authenticating on behalf of a user, so you will need the user's email address.  We'll use **username@example.com**.

A valid authentication request should look like this:

```
POST http://cloud.firebrandtech.com/api/v2/auth HTTP/1.1
X-Fcs-ApiKey: apikey123
X-Fcs-App: fcs
content-type: application/x-www-form-urlencoded
host: cloud.firebrandtech.com
Content-Length: 102

ClientId=apikey123&ClientSecret=secret789&UserName=username%40example.com
```

The response will look like this:

```
HTTP/1.1 302 Redirect
Cache-Control: private
Cache-control: no-cache="set-cookie"
Content-Type: application/json; charset=utf-8
Date: Wed, 28 Oct 2015 19:04:17 GMT
P3P: CP="CAO PSA OUR"
Set-Cookie: fcs-ticket=OTliNzY0ZDMtOTQyYS00YTNkLWE2OGQtYTRkNzAwY2QyYzQ2fHVzZXJuYW1lQGV4YW1wbGUuY29tfCB8; path=/
Set-Cookie: fcs-token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJwcmUiOiJmY3MiLCJhcHAiOiJhYmMxMjMiLCJzdWIiOiJ4eXo3ODkiLCJ1c3IiOiJ1c2VybmFtZUBleGFtcGxlLmNvbSIsImV4cCI6MTQ0NjY2Mzg1OCwic2lkIjoiT1ItRUh1VFRhMFMxdktBaTRuem8ifQ.CjaMlFgLTENe2-mLbKwRqZ22CKUkOMUlu6yiVBlrO8o; path=/
X-Fcs-Ticket: OTliNzY0ZDMtOTQyYS00YTNkLWE2OGQtYTRkNzAwY2QyYzQ2fHVzZXJuYW1lQGV4YW1wbGUuY29tfCB8
Content-Length: 0
```

A successful authentication response will include some Set-Cookie headers which you will need to parse.  The important one is:

```
Set-Cookie: fcs-token=...
```

When you receive this cookie, you should set it for `yourdomain.com` (where you are hosting the library widget) and also set it on `cloud.firebrandtech.com`.  This cookie will authenticate API calls made by the widget as well as calls to the order API described below.

Also note that your authentication request should be sent server-side, so that your secret is not shared with the client.  

## The Order API

Here are the steps required to register an ebook order with Firebrand:

- If you have not already, use the Authentication API to get the fcs-token cookie.  You will need to send this cookie when hitting cloud.firebrandtech.com.
- Construct a JSON object like this:
```
    {
        totalPrice: 12.34, // the price in US dollars
        items: [{
            ean: '9780671041045',   // the ISBN (required)
            title: "test title",    // optional
            author: 'test author',  // optional
            price: 12.34,
            quantity: 1
            } // , ... You can include additional items here as needed
        ],
        referenceId: 999, // optional - If you have assigned an order ID within your system, send that as the referenceId.
        user: {
            email: username@example.com // the order will be created for this account
        }
    }
```
- Serialize this object (e.g., with JSON.stringify())
- POST the serialized object to `https://cloud.firebrandtech.com/api/v2/orders` (be sure to send the fcs-token cookie!)
- The response will list all of the items that were added in the order, like this:
```
{
        "id":"e2332fd7-1a82-42ac-a435-a53f0131526c",
        "items":[
            {
                "id":"a57547a4-9d16-495a-8f8f-a53f0131526c",
                "ean":"9780671041045",
                "title":"Star Trek: The Dominion Wars: Book 1",
                "author":"",
                "price":12.34,
                "subTotal":0,
                "quantity":1
            }
        ],
        "user":{
            "email":"username@example.com",
            "id":"99b764d3-942a-4a3d-a68d-a4d700cd2c46"
        }
    }
```
- If you send a user email that is not recognized, an account will be created for that user.
- If you send an ISBN (in the "ean" property) that is not recognized, it will be ignored.  If you see a response with an empty list of items, this is most likely the reason.

## Displaying the Library Widget

Remember that the library widget will not work if you have not set the fcs-token cookie for the domain that is hosting the widget (yourdomain.com).  See the Authentication API section for details on this.

Here is a sample page which displays the library widget:

```
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Simon and Schuster</title>
    <meta http-equiv="P3P" content='CP="CAO PSA OUR"'>
    <link href="https://cloud.firebrandtech.com/css/widgets.css" rel="stylesheet">
    <link href="https://cloud.firebrandtech.com/css/vendor.css" rel="stylesheet">
    <link href="https://cloud.firebrandtech.com/Style/widgets.css" rel="stylesheet">
</head>
<body data-ng-app="widgets" data-fc-api="apikey123">
    <div id="ng-app">
            <div id="page" class="main-container col1-layout">
                <div class="main">
                    <!-- Place this on your page where you want the widget to display -->
                    <div data-ng-app="widgets" data-fc-library="view:table"></div>
                </div>
            </div>
     <script src="https://cloud.firebrandtech.com/widgets/scripts/fcw.js"> </script>
    </div>
</body>
</html>
```

The widget can also be integrated with your site's HTML.  Using the example above, make sure you have all of the required CSS and JS includes, and then you can add something like this to your HTML:

`<div data-ng-app="widgets" data-fc-api="apikey123" data-fc-library="view:table"></div>`

This produces a tabular list view of the user's library.  To display a grid view which includes book cover images, use this:

`<div data-ng-app="widgets" data-fc-api="apikey123" data-fc-library="view:grid"></div>`
