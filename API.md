Hushfile Server API
===================

POST /
------
A POST to `/` initiates a new file that will be uploaded to hushfile. The data with the POST command should be of content-type `application/json` or `multipart/form-data` and contain the following fields to be valid:

- `expires` (optional) - Unix timestamp indicating when the file expires and the server should remove it.
- `limit` (optional) - The number of times the file may be downloaded before being deleted.

The server will respond with HTTP <code>200 OK</code> if the file upload is successfully initiated. The content is of type `application/json` with the following fields:

- `id` - The unique identifier for the file used for subsequent requests
- `key` - A password used for subsequent file operations.

This request is called *the inital `POST` request* throughout this document.


PUT /{id}
---------
Modify file configuration before the file upload is complete.

The endpoint takes the same arguments as `POST /`, however the `key` returned by the initial `POST` request is required as a querystring parameter.

If the file configuration is updated HTTP <code>200 OK</code> is returned.

If the key is not present HTTP <code>400 Bad Request</code> is returned.

If the key does not match HTTP <code>401 Unauthorized</code> is returned.


PUT /{id}/metadata
------------------
(Should this in fact be a dictionary, where the encrypted metadata is a field as to not "hack" the api for making some metadata fields public?)

Adds or updates a metadata file to `{id}`.

The `PUT` request must be accompanied by a querystring parameter `key`.

If the upload succeeds, the server responds with HTTP <code>200 OK</code>.

If the upload has not been initialized HTTP <code>412 Precondition Failed</code> is returned.

If the key is not present HTTP <code>400 Bad Request</code> is returned.

If `key` does not match HTTP <code>401 Unauthorized</code> is returned.

If this is not added, the file will be interpreted as `text/plain`.


PUT /{id}/{index}
-----------------
When the upload has been initiated, chunks are `PUT` to the server with content-type `application/octet-stream`. `{id}` is the `id` value returned by the inital `POST` request. `{index}` is a natural number specifying the order of the chunk in relation to the other chunks.

The `PUT` request must be accompanied with a querystring parameter named `key` with the value recieved by the initial `POST` request to `/`.

If the upload succeeds, HTTP <code>200 OK</code> is returned.

If the `key` does not match the key, HTTP <code>401 Unauthorized</code> is returned.

If the key is not present HTTP <code>400 Bad Request</code> is returned.

If a `PUT` is made to a file that is complete HTTP <code>409</code> is returned.

If the file has not been initiated, HTTP <code>412 Precondition Failed</code> is returned.


POST /{id}/complete
-------------------
A `POST` request to `/{id}/complete` is finishes the file upload if a series of conditions are met. The content-type of the request is either `application/json` or `multipart/form-data` containing the following fields:

- `chunks` - The number of chunks uploaded to the server.

The `POST` request must be accompanied by a querystring parameter `key`.

If the following conditions are met, the server responds with a <code>200 OK</code>:

1. The chunks specified in the `chunks` list have all been uploaded.
2. The uploadpassword file is removed to avoid any addition or alteration of the chunks.

If the `key` does not match the password set by the server, HTTP <code>401 Unauthorized</code> is returned.

If the key is not present HTTP <code>400 Bad Request</code> is returned.

If a `PUT` is made to a file that is complete HTTP <code>409 Conflict</code> is returned.

If the file has not been initiated, HTTP <code>412 Precondition Failed</code> is returned.


GET /{id}
---------
If the file `{id}` exists, the server responds with HTTP <code>200 OK</code>. Otherwise it returns HTTP <code>404</code>.


GET /{id}/info
--------------
If the file `{id}` exists a response with HTTP <code>200 OK</code> is returned with content-type `application/json` containing the fields:

What about the size of the chunks? put them inside encrypted metadata?

- `chunks` - The number of chunks the file consists of. (Should this in fact be an array with chunk file sizes?)
- `size` - Total size of all the chunks in bytes.
- `complete` - `true` if the upload has been finished, `false` if not
- `ip` - A list of all IPs that participated in uploading file `{id}`.
- `expires` - The UNIX timestamp indicating when it will expire. (I'm not sure this should be UNIX)
- `limit` (conditional)  - The total number of times the file can be downloaded in total. (Only present if `limit` was specified at file creation)
- `downloads` (conditional) - The number of times the file has been downloaded so far. (Only present if `limit` was specified at file creation)

If the file does not exist a response with HTTP <code>404 Not Found</code> is returned.

Note that it is possible to retrieve information about files as soon as they are created, but not yet completed.


GET /{id}/metadata
------------------
If the file exists, the response is HTTP <code>200 OK</code>. The response is of type `application/octet-stream` (is this true?) containing the encrypted metadata file.

(Return type depends a bit on the arguments that `POST` takes)


GET /{id}/{index}
-----------------
In order to obtain chunks a GET request is made. To download a chunk, only the `{id}` and `{index}` values are needed.

If the file and chunks is found, a response with HTTP <code>200 OK</code> is returned. The content-type is `application/octet-stream` and contains the encrypted file.

If the chunk is not without the range specified by `/{id}/info`, the server returns HTTP <code>416 Requested Range Not Satisfiable</code>.

(What to do if the file is not found at all? 500 Internal Server Error?)


DELETE /{id}
------------
Deletes all files relating to file `{id}` on the server.

The request must be accompanied by a `key` querystring parameter matching the key specified in the initial `POST` request for that file.

On success the server responds with HTTP <code>200 OK</code>.

If `key` is wrong then the response is HTTP <code>401 Unauthorized</code> and no files are changed on the server.

If the key is not present HTTP <code>400 Bad Request</code> is returned.

(Should subsequent requests now return <code>410 Gone</code> instead of <code>404 Not Found</code> from now on?)

GET /info
-------------------
`/info` This returns a json object with the following elements, which are all defined in the server config:

- `server_operator_email` (Server operator email)
- `max_retention_hours` (The maximum number of hours a file will be kept on the server)
- `max_chunksize_bytes` (The maximum permitted size of each chunk)
- `max_filesize_bytes` (The maximum permitted total filesize)
