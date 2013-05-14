hushfile
========

hushfile is a file sharing service with clientside encryption. Idea and initial code by Thomas Steen Rasmussen / Tykling, 2013.

This code is released under 2-clause BSD license, so you are free to use this code to run your own hushfile service, modify the code, or whatever you like.

Theory of Operation
-------------------
The hushfile server is pretty simple, most of the hard work is done by the clients. The server basically takes requests and replies with HTTP status codes and json blobs. The server has an upload function which returns a fileid, the rest of the operations require an existing fileid.

The following sections describe the server API and the client functions. The details are probably only relevant if you are curious or if you are implementing a new client.

Server API
==========

Uploading
---------
    /api/upload
A POST to /api/upload uploads a new file to hushfile. The POST should contain the following fields to be valid:
- cryptofile (encrypted binary blob containing the actual file data)
- metadata (encrypted binary blob containing info on the file, filename, size, mime type, and the deletepassword.
- deletepassword (text field with the password used to delete the file from hushfile)

The server returns a json response with two fields, "status" and "fileid". If everything went well, status is "OK" and fileid contains the unique hushfile id assigned to this upload. If something goes wrong during upload, fileid is empty, and the status field contains a message explaining what went wrong. Clients can choose to show this info to the user if so desired.

After a successful upload, the server has three files saved: the encrypted file, the encrypted metadata, and a serverdata.json which is not encrypted, which contains the cleartext deletepassword and uploader ip in json format. The same password is present in the encrypted metadata, this makes it possible for the server to "authenticate" file deletions so only users with the correct file password can delete a given file.

Downloading etc.
----------------
All these functions take a fileid as parameter to work. If the specified fileid does not exist, the server returns a HTTP 404 and a json blob with two elements, fileid and exists=false. This is checked before the functions below are called, so they all operate on existing valid fileids.

    /api/exists?fileid=abcdef
This returns a json blob with two elements, fileid and exists=true for a valid fileid.

    /api/file?fileid=abcdef
This request returns the encrypted filedata for the specified fileid.

    /api/metadata?fileid=abcdef
This request downloads the encrypted metadata for the specified fileid

    /api/delete?fileid=abcdef&deletepassword=blah
This request checks if the deletepassword is valid. If it is valid, the given fileid is deleted (filedata, metadata and serverdata are all deleted), and the server then returns a json blob containing two fields fileid and deleted=true. If the deletepassword is incorrect the server returns HTTP 401 and a json blob with two fields, fileid and deleted=false.

    /api/ip?fileid=abcdef
This request returns a jsob blob with two fields, fileid and uploadip. Note that it does not require a valid password to get the uploader ip of a given fileid. The authors are aware of this and regard it as a feature.

Other requests
--------------
The server replies to all other requests with a HTTP 400 and a json blob with two fields, fileid and status="bad request".


Client operation
================
A client has two basic jobs: uploading and downloading. This section describes how a client operates, based on the hushfile/hushfile-web client.


Uploading
----------
1. A file is selected by the user
2. The client should check the file size against the server maxsize API function, and display an error if it is too large.
3. The client must generate two passwords, one for the encryption, and one deletepassword. It should be possible but difficult for a user to choose a custom password. A long automatically generated alphanumeric password is best.
4. The client must encrypt the file and base64 encode the data.
5. The client must generate a json object containing four fields:
    `{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`
6. The client must encrypt the json with the same password as the filedata, and base64 encode that as well.
7. The client must do a HTTP post to `https://servername/api/upload` with three fields: `cryptofile`, `metadata` and `deletepassword`. 
8. The client will get a HTTP 200 response with a JSON body which contains two fields. A couple of examples of responses: This is after a successful upload: `{"fileid": "51928de7aba77", "status": "OK"}` - This is after a failed upload where the server could not write the encrypted file: `{"fileid": "", "status": "unable to write cryptofile"}`
9. The client must finally present a link to the user, in the format `https://servername/fileid#password`


Downloading
------------
1. A fileid is supplied by the user
2. The client checks if the fileid is valid on the server, by querying `https://servername/api/exists?fileid=51928de7aba77`
3. If the fileid is valid the server will reply with a HTTP 200 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": true}`
4. If the fileid is invalid the server will reply with a HTTP 404 with a JSON body like so: `{"fileid": "51928de7aba77", "exists": false}`
5. If a password was not supplied, the client must ask the user for a password
6. The client must then download the metadata from the server by querying `https://servername/api/metadata?fileid=51928de7aba77`
7. The client must then decrypt the metadata using the supplied password
8. If the data cannot be decrypted or the decrypted data is not valid JSON, the client must display an error to the user and ask for a new password, and try decrypting again.
9. The decrypted JSON metadata contains four fields and could look like this: `{"filename": "secretfile.txt", "mimetype": "text/plain", "filesize": "532", "deletepassword": "vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe"}`
10. The client must now get the IP address of the uploader by sending a request to `https://servername/api/ip?fileid=51928de7aba77`
11. The client must present the metadata and IP to the user, and give the user two options, download and delete.
12. If the user wants to delete the file, the client can do that by sending a request to `https://servername/api/delete?fileid=51928de7aba77&deletepassword=vK36ocTGHaz8OYcjHX5voD8j3MgsGkg8JAXAefqe`. If the deletion is successful the server responds with a HTTP 200 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": true}`
If the deletion fails because the deletepassword is incorrect, the server responds with a HTTP 401 and a JSON object like this: `{"fileid": "51928de7aba77", "deleted": false}`
13. If the client wants to download the file, the client can get the data by sending a request to `https://servername/api/file?fileid=51928de7aba77`
14. The client must then decrypt the file with the same password as the metadata was decrypted with.
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