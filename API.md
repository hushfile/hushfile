Hushfile Server API - Uploading
================================

/api/upload
-----------------
`/api/upload`
A POST to `/api/upload` uploads a new file to hushfile, or uploads a new chunk of an existing file. 

If uploading a new file the POST should contain the following fields to be valid:
- `cryptofile` (encrypted binary blob containing the actual file data)
- `metadata` (encrypted binary blob containing info on the file: filename, size, mime type, and deletepassword.
- `chunknumber` (the number of the chunk being uploaded, 0 if the file is not being uploaded in multiple parts)
- `finishupload` (if set to true the upload will be marked as finished)
- `deletepassword` (OPTIONAL: text field with the password used to delete the file from hushfile, only included if the file should be deletable)

If uploading a new part (chunk) of an existing file, the fields `metadata` and `deletepassword` must be omitted, and the POST must additionally contain:
- `fileid` (textfield with the hushfile id of the file being uploaded)
- `uploadpassword` (textfield with the uploadpassword)

If this is the first chunk being uploaded, the server returns a json response with the following fields: 
- `status` (If everything went well, status is "OK", otherwise status contains an error message describing what went wrong)
- `fileid` (Contains the unique hushfile id assigned to this upload)
- `chunks` (The current number of chunks this file consists of, note that it is up to the client to keep track of which parts of the file have already been uploaded. The server does not care what order the chunks are uploaded in, the chunks are numbered according to the chunknumber value supplied by the client.)
- `totalsize` (The encrypted size in bytes of all the currently uploaded chunks combined)
- `finished` (true if the upload has been finished, false if not)
- `uploadpassword` (empty if finished=true, otherwise contains the password to upload additional chunks)

For each additional chunk uploaded the server returns a json response with the same fields as above, except the `uploadpassword` field is omitted.

After a successful but still unfinished upload, the server has the following files saved on the filesystem:
- `cryptofile.N` (the encrypted file(s), numbered from 0-N parts)
- `metadata` (the encrypted metadata)
- `serverdata.json` (unencrypted json file which contains the fields `deletepassword` and `uploadip`)
- `uploadpassword` (contains the uploadpassword)

After an upload has been finished, the file `uploadpassword` will be deleted, in other words only unfinished uploads have the file `uploadpassword` on the servers filesystem.

For deletable files the `deletepassword` is present in both `serverdata.json` and in the encrypted metadata, this makes it possible for the server to authenticate file deletions. This means that only users with the correct file password, which in turn can unlock the correct `deletepassword`, can delete a given fileid.


/api/finishupload
-------------------------
`/api/finishupload`
A POST to `/api/finishupload` finishes an upload, meaning that no more parts of this file will be uploaded. The POST must contain the following fields:
- `fileid` (the ID of the upload being finished)
- `uploadpassword` (The `uploadpassword` that was returned when the first chunk of this fileid was uploaded)

The server returns a json response with the following fields: 
- `status` (If everything went well, `status` is `OK`, otherwise `status` contains an error message describing what went wrong)
- `fileid` (Contains the unique id assigned to this upload) 
- `chunks` (The current number of chunks this file consists of)
- `totalsize` (The encrypted size of all the chunks combined)
- `finished` (true if the upload has been finished, false if not)
- `uploadpassword` (always empty when `finished` is `true`)

When `/api/finishupload` is called the server will check if any chunks are missing, based on the names of the files. If `cryptofile.0` and `cryptofile.2` are there, but there is no `cryptofile.1` then the server must refuse to mark the upload as finished.

Hushfile Server API - Downloading
===============================
These functions take a fileid as parameter to work. If the specified fileid does not exist, the server returns a HTTP 404 and a json blob with two elements, fileid and exists=false. This is checked before the functions below are called, so they all operate on existing valid fileids.

/api/file
--------------
`/api/file?fileid=abcdef&chunknumber=N`
This request returns the encrypted filedata for the specified fileid, chunk number N, or HTTP <code>416 Requested Range Not Satisfiable</code> and a JSON error if the chunk doesn't exist in the following format: {"fileid": "blah", "status": "Chunk number N does not exist"}

/api/metadata
----------------------
`/api/metadata?fileid=abcdef`
This request downloads the encrypted metadata for the specified fileid.

Hushfile Server API - Other API Functions
======================================
These functions take a fileid as parameter to work. If the specified fileid does not exist, the server returns a HTTP 404 and a json blob with two elements, fileid and exists=false. This is checked before the functions below are called, so they all operate on existing valid fileids.

/api/delete
-----------------
`/api/delete?fileid=abcdef&deletepassword=blah`
This request checks if the `deletepassword` is valid. If it is valid, the given fileid is deleted (filedata, metadata and serverdata are all deleted), and the server then returns a json blob containing two fields fileid and deleted=true. If the `deletepassword` is incorrect the server returns HTTP 401 and a json blob with two fields, fileid and deleted=false.

/api/ip
-----------
`/api/ip?fileid=abcdef`
This request returns a jsob blob with two fields, `fileid` and `uploadip`. `uploadip` is a comma seperated list of all IPs that have uploaded one or more chunks of the given `fileid`. Note that it does not require a valid password to get the uploader ips of a given fileid. This is a feature.

/api/exists
----------------
`/api/exists?fileid=abcdef`
This returns a json blob with the following elements:
- `fileid`
- `exists` (true for a valid fileid, false for invalid)
- `chunks` (how many chunks this upload consists of, or 0 for an invalid fileid)
- `totalsize` (total size of all the chunks in bytes, or 0 for an invalid fileid)
- `finished` (true if the upload has been finished, false if not)


Other requests
===============
The server answers all other requests with a HTTP 400 and a json blob with two fields, fileid and status="bad request".
