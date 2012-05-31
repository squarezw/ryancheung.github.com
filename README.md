# PFA Server Api V3 Introduction

This API doc is based the version *v1*. As the app iterate continuously,
the old API design can't satisfy the increasingly requirements. So it
must need some more refactor and refinement.

This time the API is designed with full RESTfull style. And the development
is driven with BDD methodology. But the old version API will still serve for old
version PFA app.

# URL & Versioning

There're two strategies for API versioning:
  1. Including a version in URLs, e.g. /api/v2/accounts/login
  2. Adding a version in HTTP Accept Headers for versioning

In this version only the first strategy is applied for API versioning. So
all of the API endpoint will be prefix with `/api/v2/`, for example:

    /api/v2/accounts/register

## HTTP Status Codes

The pfa server API attempts to return appropriate
[HTTP status code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
for every request.

## Error Messages

When the API returns error messages. it does so in json format like this:

    { "errors": [{ "message": "Invalid token", code: 4 }] }

# API Definitions

## POST accounts/register

User account registration. For the simplicity of UE, user will be registered
with identity of his device but not username and password.

  * Requires Authentication: No
  * Response Formats: json
  * HTTP Methods: POST

### Resource URL

http://moya.boohee.com/api/v2/accounts/register

### Parameters

* device_id(Required)
   
  unique id of a ios device

* name(Required)
    
  nick name of user

*    gender(Required)
     1 stands for female
     2 stands for male
     Example Values: 1

1.  This is a list item with two paragraphs. Lorem ipsum dolor
    sit amet, consectetuer adipiscing elit. Aliquam hendrerit
    mi posuere lectus.

    Vestibulum enim wisi, viverra nec, fringilla in, laoreet
    vitae, risus. Donec sit amet nisl. Aliquam semper ipsum
    sit amet velit.

2.  Suspendisse id sem consectetuer libero luctus adipiscing.

*   birthday(Required)
    birthday of user
    Example Values: 1988-08-22
*   height(Required)
    height of user, units: cm
    Example Values: 178

*   weight(Required)
    weight of user, it's a number with 1 deciaml point
    Example Values: 68.5

### Example Request

POST http://moya.boohee.com/api/v2/accounts/register

POST Data

    device_id=f67fdon76xcv90f7dsf8788fd8
    name=brent
    gender=2
    birthday=1984-12-06
    height=178
    weight=68.5

Response

    {
      "user_id": 1,
      "assessment": "",
      "token": "7vfd9f6d"
    }

### Error Codes

* 1 此设备已注册
