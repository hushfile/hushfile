hushfile
========
hushfile is a file sharing service with clientside encryption. Idea and initial code by Thomas Steen Rasmussen / Tykling, 2013.

This code is released under 2-clause BSD license, so you are free to use this code to run your own hushfile service, modify the code, or whatever you like.

Theory of Operation
====================
The hushfile server is pretty simple, most of the hard work is done by the clients. The server basically takes requests and replies with HTTP status codes and json blobs. The server has an upload function which returns a fileid, the rest of the operations require an existing fileid.

The following describes the client workflow. The details are probably only relevant if you are curious or if you are implementing a new client.

A client has two basic jobs: uploading and downloading. This section describes how a client operates, based on the hushfile/hushfile-web client. If you are implementing a new client read the API file for details.

Uploading
=============
###### 1. Pick file
A file is selected by the user.

###### 2. Check max file size
The client must check the file size against the maxsize value returned by the serverinfo API function, and display an error if it is too large.

###### 3. Decide on chunking
The client should decide whether or not to use chunking, possibly have the user decide. The serverinfo API function returns a maxchunksize value which should be taken into consideration.

###### 4. Generate passwords
The client must generate two passwords, one for the encryption, and one deletepassword. It should be possible but difficult for a user to choose custom passwords. Long automatically generated alphanumeric password are the most secure.

###### 5. Generate metadata json
The client must generate a json object containing four fields:
	`{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`

###### 6. Encrypt metadata json
The client must encrypt the json with the same password as the filedata, and base64 encode that as well.

###### 7. Encrypt first file chunk
The client must encrypt the first chunk of the file and base64 encode the data. If the servers permitted chunksize and the clients desired chunksize are both equal to or larger than than the total filesize there will only be one chunk.

###### 8. Upload first chunk and metadata
The client must do a HTTP post to `https://servername/api/upload` with five fields: `cryptofile`, `metadata`, `deletepassword`, `chunknumber` and `finishupload`. The number of the first chunk is `0`. The `finishupload` field must be set to true if this is the only chunk, and false if there are additional chunks. The metadata must be uploaded with the first chunk.

###### 9. Receive response with fileid
The client will get a HTTP 200 response with a JSON body which contains five fields:
`{"fileid": "51928de7aba77", "status": "OK", "chunks": "1", "totalsize": "12345", "finished": false}`

###### 10. Encrypt and upload remaining chunks
If any chunks remain the client must encrypt and upload each one with a HTTP post to `https://servername/api/upload` with three fields: `cryptofile`, `chunknumber` and `finishupload`. This can be done in parallel if desired. When the last chunk is uploaded the `finishupload` field should be set to true, or the upload can be marked as finished by using the seperate `finishupload` API call.

###### 11. Present link to the user
The client must finally present a link to the user, in the format `https://servername/fileid#password`


Downloading
===============
###### 1. Get hushfile URL
A hushfile URL is supplied by the user

###### 2. Check URL and fileid validity
The client checks if the fileid is a valid and finished upload on the server, by querying `https://servername/api/exists?fileid=51928de7aba77`

###### 3. If fileid is valid
If the fileid is valid and the upload is finished the server will reply with a HTTP 200 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": true, "chunks": 2, "totalsize": 123432, "finished": true}`

###### 4. If fileid is valid but unfinished
If the fileid is valid but the upload has not been finished the server will reply with a "HTTP 412 Precondition Failed" with a JSON body like so: `{"fileid": "51928de7aba77", "exists": true, "chunks": 2, "totalsize": 123432, "finished": false}`

###### 5. If fileid is invalid
If the fileid is invalid the server will reply with a HTTP 404 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": false, "chunks": 0, "totalsize": 0, "finished": false}`

###### 6. If password is missing
If a password was not supplied, the client must ask the user for a password.

###### 7. Download metadata
The client must then download the metadata from the server by querying `https://servername/api/metadata?fileid=51928de7aba77`

###### 8. Decrypt metadata
The client must then decrypt the metadata using the supplied password.

###### 9. If decrypt fails
If the data cannot be decrypted or the decrypted data is not valid JSON, the client must display an error to the user and ask for a new password, and try decrypting again.

###### 10. Metadata json
The decrypted JSON metadata contains four fields and could look like this: `{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`

###### 11. Get uploader IP
The client must now get the IP address of the uploader by sending a request to `https://servername/api/ip?fileid=51928de7aba77`

###### 12. Present download and delete options to user
The client must present the metadata and IP to the user, and give the user two options, download and delete.

###### 13. To delete file
If the user wants to delete the file, the client can do that by sending a request to `https://servername/api/delete?fileid=51928de7aba77&deletepassword=vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe`. 

If the deletion is successful the server responds with a HTTP 200 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": true}`

If the deletion fails because the deletepassword is incorrect, the server responds with a HTTP 401 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": false}`

###### 14. To download a chunk
If the client wants to download the file, the client can get the data for a given chunk by sending a request to `https://servername/api/file?fileid=51928de7aba77&chunknumber=N` - if there is only one chunk the number is `0`

###### 15. Decrypt downloaded data
The client must then decrypt the file with the same password as the metadata was decrypted with.

###### 16. Repeat for every chunk
Repeat for every chunk, put the chunks together when all of them are downloaded.

###### 17. Prompt user to save file
Finally the client can prompt the user to save the file somewhere.

###### 18. Delete option
The client should still have an active delete button after download, so a user can delete a file after downloading.


3rd Party Components
=====================
The following components are used extensively in the code:
- http://code.google.com/p/crypto-js/
	Crypto-JS is used to perform encryption and decryption.

- http://twitter.github.io/bootstrap/
	Twitter Bootstrap is used for styling

- http://fortawesome.github.io/Font-Awesome/
	Font-Awesome is a great icon collection for use with Twitter Bootstrap
