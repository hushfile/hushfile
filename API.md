# Hushfile Server API

## Content Types

If nothing else is stated explicitly, the server should accept requests with the `Content-Type` header of either `application/json` as well as `application/x-www-form-urlencoded` interchangeably and without differing in behaviour.

The server responds to requests with `application/json`.
When the server responds with <code>200 OK</code> the content type is `application/json`. The fields contained in the response will be described where needed. Otherwise the response is an empty object `{}`.

In case of status errors, the default error pages of the webserver application is used.


## Standard responses and requirements.

### 400 Bad Request
If the `Content-Type` is wrong, or the request cannot be parsed, HTTP <code>400 Bad Request</code> is returned.

### 401 Unauthorized
If `key` is required for the request as a querystring parameter, HTTP <code>401 Unauthorized</code> is returned if the `key` is not present.

### 403 Forbidden
If `key` is required for the request as a querystring parameter, HTTP <code>403 Forbidden</code> is returned if the `key` from the request does not match the one on the server.

### 404 Not Found
All requests with `{id}` and/or `{index}` as part of the path requires that the file/index exists. If the file `{id}` and/or `{index}` is not found on the server, HTTP <code>404 Not Found</code> is returned.

If an endpoint with a pattern that is not described in the API is requests, HTTP <code>404 Not Found</code> is returned.
This is also the case, if an endpoint of a pattern which is not described in the API is requested.

### 412 Precondition Failed
If the request requires that the file upload for `{id}` is not complete, HTTP <code>412 Precondition Failed</code> is returned if file `{id}` is marked as complete.


## API Endpoints

### POST /
Initiates a file upload to the server.

The server accepts the following fields in the request:

- `expires` (optional) - Unix timestamp indicating when the file expires and the server should remove it.
- `metadata` (optional) - Initial metadata object of the same format as accepted for `PUT {id}/metadata`.

If the following conditions are met, the request succeeds:

1. The file *exists* on the server (e.g. as a directory).
2. Information about expiry and download limit is persisted.
3. Default file metadata data about the file is persisted.

If the request is successful, the response contains the following fields:

- `id` - The unique identifier for the file used for subsequent requests.
- `key` - A password used for subsequent file operations.


### PUT /{id}
Modifies the settings of the initiated file upload.

The server requires that the file upload `{id}` is initiated and not complete, as well as the querystring parameter `key`.

The server accepts the same fields as `POST /`.

If the following conditions are met, the request succeeds:

1. The modified file settings are persisted.


### GET /{id}
Returns server statistical information about file `{id}`.

The server requires that file `{id}` is initiated.

The response contains the following fields:

- `chunks` - An array containing the size of the chunks present on the server in bytes.
- `size` - Total size of all the chunks in bytes.
- `complete` - `true` if the upload has been finished, `false` if not.
- `expires` - The UNIX timestamp indicating when it will expire. (I'm not sure this should be UNIX)
- `downloads` - The number of times the file has been downloaded so far.


### DELETE /{id}
Remove every trace on the server that the file was ever there

The server requires the querystring parameter `key`.

If the following conditions are met, the request succeeds:

1. All chunks related to file `{id}` are deleted.
2. All metadata information for file `{id}` is deleted.
3. All properties or files relevant only to file `{id}` are deleted.


### POST /{id}/complete
Completes the file upload for `{id}`, disabling any changes to the file.

The server requires that the file upload `{id}` is initiated and not complete, as well as the querystring parameter `key`.

The request must contain the field:
- `chunks` - A list of the filesize in bytes for the upload chunks (in order).

If the following conditions are met, the request succeeds:

1. The file `{id}` is initiated and not complete.
2. The number of chunks specified in `chunks` are present.
3. The size of each chunk specified in `chunks` matches the respective one on the server.


### PUT /{id}/{index}
Uploads a chunk with numerical `{index}` to file `{id}`.

The server requires that the file upload `{id}` is initiated and not complete, as well as the querystring parameter `key`.

The `Content-Type` of the request should be `multipart/form-data`.

If the following conditions are met, the request succeeds:

1. `{index}` is numeric.
2. `{index}` is not larger than `max_chunks`.
3. The file is successfully persisted.

The first time a chunk is uploaded, the server responds with HTTP <code>201 Created</code>.


### GET /{id}/{index}
Downloads the encrypted chunk `{index}` for the file `{id}`.

The `Content-Type` of the response is `application/octet-stream`.


### GET /{id}/metadata
Yields the metadata information added by `PUT /{id}/metadata`.


### PUT /{id}/metadata
Adds or updates the metadata information for file `{id}`.

The server requires that file upload `{id}` is initiated and not complete, as well as the querystring parameter `key`.

The server does not require the presence of the field `metadata`, but the contents of that named field should be the encrypted metadata for the file.

If the the following conditions are met, the request succeeds:

1. The fields of the request are persisted.

If no metadata is available, the client should assume a content type of `text/plain`.


### OPTIONS /
Returns information about the servers configuration.

The response contains the following fields:

- `operator_email` - Server operator email address.
- `min_retention` - The least number of seconds the server promises to keep the file.
- `max_size` - The maximum permitted size in bytes of a chunk.
- `max_chunks` - The maximum number of chunks a file can consist of.
