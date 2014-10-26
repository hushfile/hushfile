Hushfile Server API
===================

POST /api/file
---------------
A POST to `/api/file` initiates a new file that will be uploaded to hushfile. The data with the POST command should be of content-type `application/json` or `multipart/form-data` and contain the following fields to be valid:

- `metadata` - An encrypted binary blob containing the information about the file, such as filename, size, mime-type. It can also contain an optional deletepassword field.
- `mac` - A Message Authentication Code (MAC) for the unencrypted metadata object.
- `deletepassword` (optional) - A text field containing a password that the client will use in order to force removal of the file on the server.
- `expire` (optional) - Unix timestamp indicating when the file expires and the server should remove it.
- `limit` (optional) - The number of times the file may be downloaded before being deleted.
- `chunks` (optional) - The number of chunks that the file is partitioned in. If it is set the upload will automatically stop when this number of chunks is reached.

The server will respond with a 200 status code if the file upload is successfully initiated. The content is of type `application/json` with the following fields:

- `id` - The unique identifier for the file used for subsequent requests
- `uploadpassword` - The initial password that should be used for subsequent uploads of chunks

PUT /api/file/{id}
-------------------
When the upload has been initiated, chunks are PUT to the server with content-type `application/json` or `multipart/form-data` with the following fields:

- `uploadpassword` - The password that was returned by the POST to `/api/file/`.
- `index` - The index number of the chunk.
- `chunk` - The encrypted blob that contains the file data of the chunk.
- `mac` - The MAC code for the unencrypted version of the chunk.
- `finishupload` - JSON boolean, if set to true, the upload will be marked as finished.

If `finishupload` is true, the server checks whether chunks {id}.0, ..., {id}.N exists, if not, the upload is not finished.

If the upload succeeds, a status code of 200 is returned. The body of the response is `application/json` and contains the following fields:

- `index` - The index that the server recorded.
- `finished` - JSON boolean indicating whether the upload is now finished.

If the `uploadpassword` does not match the password set by the server a response with status code 403 is returned.

GET /api/file/{id}/exists
--------------------
If the file `id` exists, the repsonse status code is 200. If it does not exist the status code is 404.

GET /api/file/{id}/info
------------------
If the file exists a response with HTTP <code>200 OK</code> is returned with content-type `application/json` containing the fields:

- `chunks` - how many chunks this upload consists of, or 0 for an invalid fileid
- `totalsize` - total size of all the chunks in bytes, or 0 for an invalid fileid
- `finished` - true if the upload has been finished, false if not
- `ip` - A list of all IPs that participated in uploading file {id}.
- `expire` - The UNIX timestamp indicating when it will expire.
- `limit` - The total number of times the file can be downloaded.
- `downloads` - The number of times the file has been downloaded.

If the file does not exist a response with HTTP <code>404 Not Found</code> is returned.

GET /api/file/{id}/metadata
---------------------------
If the file exists, the response has status code 200 OK. The response is of type `application/json` and contains the following fields:

- `metadata` - The encrypted metadata blob
- `mac` - The MAC of the unencrypted metadata object

If the upload is unfinished the server must instead return HTTP <code>412 Precondition Failed</code> and a json error in the following format: {"fileid": "blah", "status": "Upload is unfinished."}

GET /api/file/{id}/{index}
--------------------------
In order to obtain chunks a GET request is made. To download a chunk, only the `id` and `index` is needed. If the file and chunks is found, a response with status code 200 is returned. The content-type is `application/json` and contains the following fields:

- `chunk` - The encrypted blob of the chunk
- `mac` - The MAC of the unencrypted chunk

If the upload is unfinished the server must instead return HTTP <code>412 Precondition Failed</code> and a json error in the following format: {"fileid": "blah", "status": "Upload is unfinished."}

HTTP <code>416 Requested Range Not Satisfiable</code> and a JSON error if the chunk doesn't exist.


DELETE /api/file/{id}/{deletepassword}
--------------------------------------
If the `deletepassword` matches the `id` or there is no deletepassword all files on the server relating to the file is deleted. The response is then <code>200 OK</code>. If the deletepassword is wrong then the response is a <code>403 Forbidden</code>.

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
