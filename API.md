Hushfile Server API
===================

POST /api/file
---------------
A POST to `/api/file` initiates a new file that will be uploaded to hushfile. The data with the POST command should be of content-type `application/json` or `multipart/form-data` and contain the following fields to be valid:

- `deletepassword` (optional) - A text field containing a password that the client will use in order to force removal of the file on the server.
- `expire` (optional) - Unix timestamp indicating when the file expires and the server should remove it.
- `limit` (optional) - The number of times the file may be downloaded before being deleted.

The server will respond with HTTP <code>200 OK</code> if the file upload is successfully initiated. The content is of type `application/json` with the following fields:

- `id` - The unique identifier for the file used for subsequent requests
- `uploadpassword` - The initial password that should be used for subsequent uploads of chunks

This request is called *the inital `POST` request* throughout this document.

PUT /api/file/{id}/metadata
---------------------------
Adds a metadata file to `{id}`. If this is not added, the file will be interpreted as `text/plain`.

The `PUT` request must be accompanied by a querystring parameter `uploadpassword`.

If the upload succeeds, the server responds with HTTP <code>200 OK</code>. If `uploadpassword` does not match the uploadpassword returned by the initial `POST` request HTTP <code>403 Forbidden</code> is returned. Otherwise HTTP <code>400 Bad Request</code> is returned.

PUT /api/file/{id}/{index}
--------------------------
When the upload has been initiated, chunks are `PUT` to the server with content-type `application/octet-stream`. `{id}` is the `id` value returned by the inital `POST` request. `{index}` is a natural number specifying the order of the chunk in relation to the other chunks.

The `PUT` request must be accompanied with a querystring parameter named `uploadpassword` with the value recieved by the initial `POST` request to `/api/file`.

If the upload succeeds, HTTP <code>200 OK</code> is returned. The body of the response is `application/json` and contains the following fields:

- `index` - The index that the server recorded.
- `finished` - JSON boolean indicating whether the upload is now finished.

If the `uploadpassword` does not match the password set by the server, HTTP <code>403 Forbidden</code> is returned.

If a `PUT` is made to a file that is complete or has not been initiated, HTTP <code>400 Bad Request</code> is returned.


POST /api/file/{id}/finishupload
--------------------------------
A request to `/api/file/{id}/finishupload` is finishes the file upload if a series of conditions are met. The content-type of the request is either `application/json` or `multipart/form-data` containing the following fields:

- `uploadpassword` - The upload password specified by the initial `POST` request.
- `chunks` - A list of chunk index numbers that the file consists of.

If the following conditions are met, the server responds with a <code>200 OK</code>:

1. The chunks specified in the `chunks` list have all been uploaded.
2. The uploadpassword is removed to avoid any addition or alteration of the chunks.

GET /api/file/{id}/exists
--------------------
If the file `{id}` exists and is completed, the server responds with HTTP <code>200 OK</code>. Otherwise it returns HTTP <code>404</code>.

GET /api/file/{id}/info
------------------
If the file `{id}` exists a response with HTTP <code>200 OK</code> is returned with content-type `application/json` containing the fields:

- `chunks` - A list of ids that the file consists of.
- `totalsize` - Total size of all the chunks in bytes, or 0 for an invalid fileid.
- `finished` - `true` if the upload has been finished, `false` if not
- `ip` - A list of all IPs that participated in uploading file `{id}`.
- `expire` - The UNIX timestamp indicating when it will expire.
- `limit` - The total number of times the file can be downloaded.
- `downloads` - The number of times the file has been downloaded so far.

If the file does not exist a response with HTTP <code>404 Not Found</code> is returned.

GET /api/file/{id}/metadata
---------------------------
If the file exists, the response is HTTP <code>200 OK</code>. The response is of type `application/octet-stream` containing the encrypted metadata file.

If the upload is unfinished the server must instead return HTTP <code>412 Precondition Failed</code> and a json error in the following format: {"fileid": "blah", "status": "Upload is unfinished."}

GET /api/file/{id}/{index}
--------------------------
In order to obtain chunks a GET request is made. To download a chunk, only the `{id}` and `{index}` values are needed. If the file and chunks is found, a response with HTTP <code>200 OK</code> is returned. The content-type is `application/octet-stream` and contains the encrypted file.

If the upload is unfinished the server must instead return HTTP <code>412 Precondition Failed</code>.

If the chunk does not exist, the server must return HTTP <code>416 Requested Range Not Satisfiable</code>.


DELETE /api/file/{id}
---------------------
Deletes all files relating to file `{id}` on the server. The request must be accompanied by a `deletepassword` querystring parameter matching the deletepassword value specified in the initial `POST` request for that file.


On success the server responds with HTTP <code>200 OK</code>. If `deletepassword` is wrong then the response is HTTP <code>403 Forbidden</code> and no files are changed on the server.

GET /api/serverinfo
-------------------
`/api/serverinfo` This returns a json object with the following elements, which are all defined in the server config:

- `server_operator_email` (Server operator email)
- `max_retention_hours` (The maximum number of hours a file will be kept on the server)
- `max_chunksize_bytes` (The maximum permitted size of each chunk)
- `max_filesize_bytes` (The maximum permitted total filesize)

Other requests
===============
The server answers all other requests with a HTTP 400 and a json blob with two fields, fileid and status="bad request".
