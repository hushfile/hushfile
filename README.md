hushfile
========

hushfile is a file sharing service with clientside encryption. Idea and initial code by Thomas Steen Rasmussen / Tykling, 2013.

This code is released under 2-clause BSD license, so you are free to use this code to run your own hushfile service, modify the code, or whatever you like.

Theory of Operation
-------------------
The hushfile server is pretty simple, most of the hard work is done by the clients. The server basically takes requests and replies with HTTP status codes and json blobs. The server has an upload function which returns a fileid, the rest of the operations require an existing fileid.

Uploading
---------
    /api/upload
A POST to /api/upload uploads a new file to hushfile. The POST should contain the following fields to be valid:
- cryptofile (encrypted binary blob containing the actual file data)
- metadata (encrypted binary blob containing info on the file, filename, size, mime type, and the deletepassword.
- deletepassword (text field with the password used to delete the file from hushfile)

The server returns a json response with two fields, "status" and "fileid". If everything went well, status is "OK" and fileid contains the unique hushfile id assigned to this upload. If something goes wrong during upload, the status field contains a message explaining what went wrong. Clients can choose to show this info to the user if so desired.

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

3rd Party Components
--------------------
The following components are used extensively in the code:
- http://code.google.com/p/crypto-js/
	Crypto-JS is used to perform encryption and decryption.

- http://twitter.github.io/bootstrap/
	Twitter Bootstrap is used for styling

- http://fortawesome.github.io/Font-Awesome/
	Font-Awesome is a great icon collection for use with Twitter Bootstrap