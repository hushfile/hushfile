Hushfile Server API
====================

Uploading
---------
	/api/upload
A POST to <code>/api/upload</code> uploads a new file to hushfile, or a new chunk of an existing file. 

If uploading a new file the POST should contain the following fields to be valid:
- <code>cryptofile</code> (encrypted binary blob containing the actual file data)
- <code>metadata</code> (encrypted binary blob containing info on the file: filename, size, mime type, and deletepassword.
- <code>deletepassword</code> (text field with the password used to delete the file from hushfile)
- <code>chunknumber</code> (the number of the chunk being uploaded, 0 if the file is not being uploaded in multiple parts)
- <code>finishupload</code> (if set to true will automatically mark the upload as finished)

If uploading a new part (chunk) of an existing file, the fields <code>metadata</code> and <code>deletepassword</code> must be omitted, and the POST must additionally contain:
- <code>fileid</code> (textfield with the hushfile id of the file being uploaded)
- <code>uploadpassword</code> (textfield with the uploadpassword)


The server returns a json response with the following fields: 
- <code>status</code> (If everything went well, status is "OK", otherwise status contains an error message describing what went wrong)
- <code>fileid</code> (Contains the unique hushfile id assigned to this upload)
- <code>chunks</code> (The current number of chunks this file consists of, note that it is up to the client to keep track of which parts of the file have already been uploaded. The server does not care what order the chunks are uploaded in, the chunks are numbered according to the chunknumber value supplied by the client.)
- <code>totalsize</code> (The encrypted size in bytes of all the currently uploaded chunks combined)
- <code>finished</code> (true if the upload has been finished, false if not)
- <code>uploadpassword</code> (null if finished=true, otherwise contains the password to upload the next chunk)

After a successful but still unfinished upload, the server has the following files saved on the filesystem:
- <code>cryptofile.N</code> (the encrypted file(s), numbered from 0-N parts)
- <code>metadata</code> (the encrypted metadata)
- <code>serverdata.json</code> (unencrypted json file which contains the fields <code>deletepassword</code> and <code>uploadip</code>)
- <code>uploadpassword</code> (contains the uploadpassword)

After an upload has been finished, the file <code>uploadpassword</code> will be deleted, in other words only unfinished uploads have the file <code>uploadpassword</code> on the servers filesystem.

The deletepassword is present in both <code>serverdata.json</code> and in the encrypted metadata, this makes it possible for the server to authenticate file deletions. This means that only users with the correct file password, which in turn can unlock the correct deletepassword, can delete a given fileid.

*Please note* It is possible for a malicious client to construct an upload where the deletepassword POSTed to the server is different from the one inside the encrypted metadata blob. This will effectively create an upload that cannot be deleted by clients who don't know the real deletepassword. This is a designflaw that the author is aware of. Suggestions for alternative approaches appreciated.

	/api/finishupload
A POST to <code>/api/finishupload</code> finishes an upload, meaning that no more parts of this file will be uploaded. The POST must contain the following fields:
- <code>fileid</code> (the ID of the upload being finished)
- <code>uploadpassword</code> (The uploadpassword that was returned when the last part of this fileid was uploaded)

The server returns a json response with the following fields: 
- <code>status</code> (If everything went well, status is "OK", otherwise status contains an error message describing what went wrong)
- <code>fileid</code> (Contains the unique hushfile id assigned to this upload) 
- <code>chunks</code> (The current number of chunks this file consists of)
- <code>totalsize</code> (The encrypted size of all the chunks combined)
- <code>finished</code> (true if the upload has been finished, false if not)
- <code>uploadpassword</code> (always null when finished=true)

When finishupload is called the server will check if any chunks are missing, based on the names of the files. If cryptofile.0 and cryptofile.2 are there, but there is no cryptofile.1 then the server must refuse to mark the upload as finished.

Downloading etc.
----------------
These functions take a fileid as parameter to work. If the specified fileid does not exist, the server returns a HTTP 404 and a json blob with two elements, fileid and exists=false. This is checked before the functions below are called, so they all operate on existing valid fileids.

	/api/file?fileid=abcdef&chunknumber=N
This request returns the encrypted filedata for the specified fileid, chunk number N, or HTTP <code>416 Requested Range Not Satisfiable</code> and a JSON error if the chunk doesn't exist in the following format: {"fileid": "blah", "status": "Chunk number N does not exist"}

	/api/metadata?fileid=abcdef
This request downloads the encrypted metadata for the specified fileid.

Other API functions
--------------------
	/api/delete?fileid=abcdef&deletepassword=blah
This request checks if the deletepassword is valid. If it is valid, the given fileid is deleted (filedata, metadata and serverdata are all deleted), and the server then returns a json blob containing two fields fileid and deleted=true. If the deletepassword is incorrect the server returns HTTP 401 and a json blob with two fields, fileid and deleted=false.

	/api/ip?fileid=abcdef
This request returns a jsob blob with two fields, fileid and uploadip, uploadip is a comma seperated list of all IPs that have uploaded one or more chunks of the given fileid. Note that it does not require a valid password to get the uploader ips of a given fileid. The authors are aware of this and regard it as a feature.

	/api/exists?fileid=abcdef
This returns a json blob with the following elements:
- <code>fileid</code>
- <code>exists</code> (true for a valid fileid, false for invalid)
- <code>chunks</code> (how many chunks this upload consists of, or 0 for an invalid fileid)
- <code>totalsize</code> (total size of all the chunks in bytes, or 0 for an invalid fileid)
- <code>finished</code> (true if the upload has been finished, false if not)


Other requests
---------------
The server answers all other requests with a HTTP 400 and a json blob with two fields, fileid and status="bad request".
