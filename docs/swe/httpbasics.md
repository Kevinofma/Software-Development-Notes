# Key Concepts in HTTP 
This page provides an overview of the fundamental concepts and structures of HTTP as used in modern web APIs. You will learn how URLs are broken down, the purpose and behavior of HTTP methods, how requests and responses are structured, key status codes, and the different ways input can be provided to an HTTP request. Additionally, REST principles and the roles of methods, paths, query parameters, bodies, and headers are explained to help you design and understand RESTful APIs.

## URL Breakdown

Example URL:  
```
https://api.instagram.com/users/123/posts/456/comments
```

Breaking down the URL:
- `https://` → protocol (HTTPS = secure HTTP)  
- `api.instagram.com` → server domain  
- `users/123` → specific user (resource)  
- `posts/456` → specific post by that user  
- `comments` → collection of comments on the post  

---

## HTTP Methods

| Method | Safety | Idempotent | Description |
|--------|--------|------------|------------|
| GET    | Safe   | Yes        | Fetches information without modifying resources |
| POST   | Not Safe | No       | Creates new resources |
| PUT    | Not Safe | Yes      | Replaces an existing resource entirely |
| PATCH  | Not Safe | Yes      | Updates part of an existing resource |
| DELETE | Not Safe | Yes      | Removes a resource |

??? Notes "Idempotency"
    * Idempotent = repeating the request has the same effect as doing it once  
    * Example: pressing a "delete account" button multiple times does not create multiple deletes (DELETE is idempotent)  
    * POST is **not** idempotent: repeated POST requests can create multiple copies  

---

## Example Uses & Common Use Cases

### **GET**
- **Examples:** Viewing a user’s profile page, retrieving a list of blog posts, fetching search results  
- **Use Cases:** Search operations, data retrieval, reading resources, querying system status  

### **POST**
- **Examples:** Adding a new comment, creating a new user account, uploading a file  
- **Use Cases:** Form submissions, file uploads, resource creation, data processing  

### **PUT**
- **Examples:** Updating an entire user profile, replacing a document, full configuration updates  
- **Use Cases:** Full resource updates, complete replacements, version management  

### **PATCH**
- **Examples:** Updating a user’s email, modifying specific fields, incremental configuration changes  
- **Use Cases:** Partial updates, field-specific modifications, resource property adjustments  

### **DELETE**
- **Examples:** Removing a comment, deleting a user account, deleting a file from storage  
- **Use Cases:** Resource removal, cleanup operations, account deletion, content management  

---

## Anatomy of API Communication

### Request (Client → Server)

```http
POST /posts/12345/comments
Content-Type: application/json
Authorization: Bearer eyJhbGc...

{
    "comment": "Great photo!",
    "timestamp": "2024-01-26T10:30:00Z",
    "source": "mobile_app"
}
```

Components:

* Method & URL: Action and resource

* Headers: Metadata (content type, auth, etc.)
    * Content-Type: Tells the server what format the data is in.
    * Authorization: Proves who you are (usually a token).

* Body: Data being sent (JSON, form data, etc.)

### Response (Server → Client)
```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-cache

{
    "success": true,
    "message": "Comment added successfully",
    "comment_id": 789,
    "timestamp": "2024-01-26T10:30:01Z"
}
```

Components:

* Status Line: HTTP version + status code + message

* Headers: Metadata (content type, cache control) 
    * Content-Type: Format of the response (JSON).
    * Cache-Control: Instructions for how clients should cache the response (or not!)

* Body: Actual response data (success confirmation, new IDs, etc.)

## Status Codes

!!! abstract "**Informational (1xx)**: Connection-level information, rarely used in APIs "
    * *101 Switching Protocols*: Used when an HTTP connection is upgraded to a WebSocket  


!!! success "**Success (2xx)**: Request successfully processed  "
    * *200 OK*: Request succeeded  
    * *201 Created*: Resource created successfully  

!!! warning "**Redirection (3xx)**: Client must take additional action "
    * *301 Moved Permanently*: Resource permanently moved to a new URL  
    * *302 Found*: Resource temporarily at a different URL  
    * *307 Temporary Redirect*: Like 302, but method must not change  

!!! failure "**Client Error (4xx)**: Issue with the client request  "
    * *400 Bad Request*: Invalid syntax or parameters  
    * *401 Unauthorized*: Authentication required or failed  
    * *403 Forbidden*: User does not have permission  
    * *404 Not Found*: Resource does not exist  

!!! bug "**Server Error (5xx)**: Server-side problem  "
    * *500 Internal Server Error*: Unexpected server error  

## Providing Input to an HTTP Request

HTTP requests can include input in several ways, each serving a specific purpose:  

- **Methods**: Define the type of action being performed (e.g., GET, POST, PUT, PATCH, DELETE).  
- **Paths**: Identify and route to specific resources.  
- **Query Parameters**: Refine or filter results without changing the resource's identity.  
- **Bodies**: Send structured data, typically for creating or updating resources.  
- **Headers**: Include metadata about the request, such as authentication or content type.

---

## REST (Representational State Transfer)

REST is an architectural style for designing HTTP APIs that focuses on resources and stateless communication. Key points:  

- Each request from client to server contains all necessary information.  
- Clients and servers can evolve independently without breaking each other.  
- RESTful APIs use URLs to identify resources and standard HTTP methods to operate on them.  

---

## Input Components

### Methods

Define the action you want to perform on a resource. Common methods:  

- **GET**: Fetch information (safe, idempotent)  
- **POST**: Create new resources (not safe, not idempotent)  
- **PUT**: Completely replace a resource (not safe, idempotent)  
- **PATCH**: Partially update a resource (not safe, idempotent)  
- **DELETE**: Remove a resource (not safe, idempotent)  


### Paths

Paths identify resources and define the structure of an API. Examples:  

- `/api/users` → Collection of users  
- `/api/users/42` → Specific user with ID 42  
- `/api/users/42/orders` → Orders of user 42  

Dynamic parts (like `42`) are variables; static parts (like `/users/`) remain constant.

### Query Parameters

Query parameters refine requests without changing the resource identity. Examples:  

- `/books?genre=fantasy&sort=newest` → Filter books by genre and sort by newest  
- Common uses: filtering, sorting, pagination, searching  

### Bodies

Bodies carry structured data in requests (mainly for POST/PUT).  

Example JSON body:  

```json
{
  "name": "Alice",
  "email": "alice@example.com"
}
```

* Defines a schema: field names, data types, and structure
* Can include complex structures like lists or nested objects

### Headers

Headers provide metadata about requests. Common uses:

* **Authorization**: e.g., `Authorization: Bearer abc123xyz`
* **Content-Type**: e.g., `application/json`
* **Accept**: Request a specific response format

Headers can also support caching, security features, and content negotiation.