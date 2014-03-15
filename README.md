hushfile
========

hushfile is a file sharing service with clientside encryption. Idea and initial code by Thomas Steen Rasmussen / Tykling, 2013.

This code is released under 2-clause BSD license, so you are free to use this code to run your own hushfile service, modify the code, or whatever you like.

Theory of Operation
====================
The hushfile server is pretty simple, most of the hard work is done by the clients. The server basically takes requests and replies with HTTP status codes and json blobs. The server has an upload function which returns a fileid, the rest of the operations require an existing fileid.

The following describes the client workflow. The details are probably only relevant if you are curious or if you are implementing a new client.

A client has two basic jobs: uploading and downloading. This section describes how a client operates, based on the hushfile/hushfile-web client. If you are implementing a new client read the API file for details.


Uploading without chunking
---------------------------
###### 1. Pick file
A file is selected by the user
###### 2. Check size
The client should check the file size against the server maxsize API function, and display an error if it is too large.
###### 3. Decide on chunking
The client should decide wether or not to use chunking, possibly have the user decide. Uploading smaller chunks of a large file will be less ressource intensive. In this case, the whole file is uploaded in one piece.
###### 4. Generate passwords
The client must generate two passwords, one for the encryption, and one deletepassword. It should be possible but difficult for a user to choose custom passwords. Long automatically generated alphanumeric password are the most secure.
###### 5. Encrypt file
The client must encrypt the file and base64 encode the data.
###### 6. Generate metadata json
The client must generate a json object containing four fields:
	`{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`
###### 7. Encrypt metadata json
The client must encrypt the json with the same password as the filedata, and base64 encode that as well.
###### 8. Upload data and metadata
The client must do a HTTP post to `https://servername/api/upload` with five fields: `cryptofile`, `metadata`, `deletepassword`, `chunknumber` (always 0 when not chunking) and `finishupload`. 
###### 9. Receive response with fileid
The client will get a HTTP 200 response with a JSON body which contains five fields:
{"fileid": "51928de7aba77", "status": "OK", "chunks": "1", "totalsize": "12345", "finished": true}
###### 10. Present link to the user
The client must finally present a link to the user, in the format `https://servername/fileid#password`

Uploading with chunking
------------------------
###### 1. Pick file
A file is selected by the user
###### 2. Check size
The client should check the file size against the server maxsize API function, and display an error if it is too large.
###### 3. Decide on chunking
The client should decide wether or not to use chunking, possibly have the user decide. Uploading smaller chunks of a large file will be less ressource intensive. In this case the file is uploaded in multiple chunks.
###### 4. Generate passwords
The client must generate two passwords, one for the encryption, and one deletepassword. It should be possible but difficult for a user to choose custom passwords. Long automatically generated alphanumeric password are the most secure.
###### 5. Encrypt file chunk
The client must encrypt the first chunk of the file and base64 encode the data.
###### 6. Generate metadata json
The client must generate a json object containing four fields:
	`{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`
###### 7. Encrypt metadata json
The client must encrypt the json with the same password as the filedata, and base64 encode that as well.
###### 8. Upload data and metadata
The client must do a HTTP post to `https://servername/api/upload` with five fields: `cryptofile`, `metadata`, `deletepassword`, `chunknumber` and `finishupload`. 
###### 9. Receive response with fileid
The client will get a HTTP 200 response with a JSON body which contains five fields:
`{"fileid": "51928de7aba77", "status": "OK", "chunks": "1", "totalsize": "12345", "finished": false}`
10. Set finishupload when done
The client repeats points 5 through 9 until no more chunks remain. The last upload POST should set `finishupload` to true or use the seperate `finishupload` API call to mark the upload as finished.
###### 11. Present link to the user
The client must finally present a link to the user, in the format `https://servername/fileid#password`

Downloading
------------
1. A hushfile URL is supplied by the user
2. The client checks if the fileid is a valid and finished upload on the server, by querying `https://servername/api/exists?fileid=51928de7aba77`
3. If the fileid is valid and the upload is finished the server will reply with a HTTP 200 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": true, "chunks": 2, "totalsize": 123432, "finished": true}`
3. If the fileid is valid but the upload has not been finished the server will reply with a "HTTP 412 Precondition Failed" with a JSON body like so: `{"fileid": "51928de7aba77", "exists": true, "chunks": 2, "totalsize": 123432, "finished": false}`
4. If the fileid is invalid the server will reply with a HTTP 404 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": false, "chunks": 0, "totalsize": 0, "finished": false}`
5. If a password was not supplied, the client must ask the user for a password
6. The client must then download the metadata from the server by querying `https://servername/api/metadata?fileid=51928de7aba77`
7. The client must then decrypt the metadata using the supplied password
8. If the data cannot be decrypted or the decrypted data is not valid JSON, the client must display an error to the user and ask for a new password, and try decrypting again.
9. The decrypted JSON metadata contains four fields and could look like this: `{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`
10. The client must now get the IP address of the uploader by sending a request to `https://servername/api/ip?fileid=51928de7aba77`
11. The client must present the metadata and IP to the user, and give the user two options, download and delete.
12. If the user wants to delete the file, the client can do that by sending a request to `https://servername/api/delete?fileid=51928de7aba77&deletepassword=vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe`. If the deletion is successful the server responds with a HTTP 200 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": true}`
If the deletion fails because the deletepassword is incorrect, the server responds with a HTTP 401 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": false}`
13. If the client wants to download the file, the client can get the data by sending a request to `https://servername/api/file?fileid=51928de7aba77&chunknumber=N`
14. The client must then decrypt the file with the same password as the metadata was decrypted with.
14. Repeat for every chunk, put the chunks together when all of them are downloaded
15. Finally the client can prompt the user to save the file somewhere.
16. The client should still have an active delete button after download, so a user can delete a file after downloading.


3rd Party Components
=====================
The following components are used extensively in the code:
- http://code.google.com/p/crypto-js/
	Crypto-JS is used to perform encryption and decryption.

- http://twitter.github.io/bootstrap/
	Twitter Bootstrap is used for styling

- http://fortawesome.github.io/Font-Awesome/
	Font-Awesome is a great icon collection for use with Twitter Bootstrap
