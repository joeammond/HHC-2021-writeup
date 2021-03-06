# LFI to RCE... maybe?

There are [many](https://www.rcesecurity.com/2017/08/from-lfi-to-rce-via-php-sessions/)
[articles](https://outpost24.com/blog/from-local-file-inclusion-to-remote-code-execution-part-1)
[written](https://hardik-solanki.medium.com/lfi-to-rce-by-injecting-access-log-ebe4bc789bef)
[about](https://book.hacktricks.xyz/pentesting-web/file-inclusion)
[abusing](https://github.com/RoqueNight/LFI---RCE-Cheat-Sheet)
[LFI](https://github.com/payloadbox/rfi-lfi-payload-list)
[vulnerabilities](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md) to achieve Remote Code/Command Execution. After reading the PHP
code that drives the application, my opinion is that it isn't possible in this instance.
If someone is able to actually get RCE in this application, I'd love to know how
it was achieved.

## Understanding the Application

The `apply.jackfrosttower.com` website is entirely driven from the single `index.html`. 
The page returned to the user is driven by the `/?p=` query string sent to the page. 
If `p` is empty, the main index page is returned. Otherwise, there are checks in
the code to look for different values of `p`, which generate the HTML for the other
pages on the site:

``` php
<?php
} elseif (isset($_GET['p']) && $_GET['p'] == 'opportunities') {
?>

(opportunities page HTML)

<?php
} elseif (isset($_GET['p']) && $_GET['p'] == 'about') {
?>

(about page HTML)

<?php
} elseif (isset($_GET['p']) && $_GET['p'] == 'apply') {
?>

(apply page HTML)
```

The HTML for these pages are in `index.html` and not read or pulled in via PHP's `include()`
function, which eliminates abusing `p` as a path to LFI or RCE. 

The PHP code that drives the actual application is earlier in `index.html`:

``` php
<?php

if (array_key_exists("submit", $_GET)) {
    // Applicant submitted application, w00t! Process data.
    $headers=array_keys($_GET);
    if (!preg_match("/^[a-zA-Z0-9' ]+$/", $_GET['inputName'])) {
      die("Invalid name input");
    }

    $filename = $_GET['inputName'] . date('Ymdhis') . ".csv"; //filename
    $file = fopen($filename, 'a');
    if ($file) {
        fputcsv($file, $headers );
        fputcsv($file, $_GET );
        fclose($file);

        // Start to display response
?>
```

The code starts by checking whether there are parameters passed in the HTTP GET request. 
Next, a check of the  `inputName` parameter performed to ensure it only contains
upper and lower case ASCII letters, or digits. If it doesn't, the program terminates. This
check is important, as the `inputName` field is later used as the output file for the
LFI/SSRF. This filter reduces the likelyhood that an attacker can manipulate the filename
generated by the application, in an effort to pivot to an RCE vulnerability.

Next, the application opens a file of `inputName` appended with a date/time stamp, and
a `.csv` extension. 
The contents of the query string values are then appended to the file. These files are accessible from the web server, as they're written
to the `/var/www/html` directory:

``` shell-session
bash-5.0# ls -l
total 32
drwxr-xr-x    2 root     root          4096 Jan  1 03:04 css
drwxrwxrwx    1 root     root          4096 Jan  7 18:00 images
-rw-r--r--    1 root     root         14167 Dec 29 15:40 index.html
-rw-r--r--    1 nobody   nobody         175 Jan  7 18:00 pugpug20220107060004.csv
-rw-r--r--    1 nobody   nobody         174 Jan  7 18:00 pugpug20220107060007.csv
...

bash-5.0# cat pugpug20220107060004.csv
inputName,inputEmail,inputPhone,inputField,resumeFile,additionalInformation,submit,inputWorkSample
pugpug,pug@pug.pug,313-555-1212,"Aggravated pulling of hair",,,,/etc/passwd

```

While the contents of these files contain user-submitted data and the filenames are
easily discoverable, abusing these to execute code isn't possible, as the web server
and PHP-FPM instance is only configured to execute `.php` and `.html` files, not
`.csv`.

## The SSRF/LFI Exploit

The actual SSRF vulnerability is in the next code block:

``` php
<?php
    } else {
        die("Unable to open file named $filename");
    }

    if(isset($_GET['inputWorkSample'])) {
        $image_url=$_GET['inputWorkSample'];
        $data = file_get_contents($image_url);
        $new = 'images/' . $_GET['inputName'] . '.jpg';
        $upload = file_put_contents($new, $data);
        if($upload) {
            //echo "<img class='rounded mx-auto d-block img-thumbnail' width='200px' src='images/" . $_GET['inputName'] . ".jpg'>";
?>
```

The call to `file_get_contents()` is the
vulnerability. The application performs no checks on
whether the input to the function is a valid URL, matches an
approved whitelist of locations, or other methods of [preventing an SSRF
attack](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
The application writes the data retrieved from
`file_get_contents()` to the file `images/[inputName].jpg` using the
`file_put_contents()` function. As we saw earlier, `inputName` is filtered
to only contain letters and numbers, eliminating any potential filename abuse.

Most LFI-RCE vulerability paths take advantage of PHP's `include()`
function, which will execute any PHP code contained in the data stream to
be included. `file_get_contents()`, however, does not interpret any PHP code
in the content returned from the opened URL or filename. The filename itself
can contain PHP filters, as we saw when using filters to return large files,
but the contents are not executed by PHP. 

Most paths of exploiting an LFI vulnerability to achieve RCE from PHP involve 
using PHP filters such as `expect://`, `zip://`, or `phar://` to execute commands. 
However, the PHP installation in the container doesn't contain support for any
of those PHP wrappers. This can be seen in the `/wwwlog/error.log` file in the
local container, after attempting a `expect://` url:

> ```2022/01/07 17:14:32 [error] 30#30: *1325 FastCGI sent in stderr: "PHP message: PHP Warning:  file_get_contents(): Unable to find the wrapper &quot;expect&quot; - did you forget to enable it when you configured PHP? in /var/www/html/index.html on line 97PHP message: PHP Warning:  file_get_contents(expect://foo): failed to open stream: No such file or directory in /var/www/html/index.html on line 97" while reading response header from upstream, client: 10.114.180.49, server: _, request: "GET /?inputName=pugpug228&inputEmail=pug%40pug.pug&inputPhone=313-555-1212&inputField=Aggravated+pulling+of+hair&resumeFile=&additionalInformation=&submit=&inputWorkSample=expect%3A%2F%2Ffoo HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "10.114.180.112:8888"```

Similar logs are generated when the other methods of RCE are attempted.

## Easter Egg, Trolling, or Old Code?

At the top of `index.html` are the following lines:

``` php
<?php
define('DB_NAME', 'intern');
define('DB_USER', 'intern');
define('DB_PASSWORD', 'polarwinds');
?>
```

The container doesn't appear to contain any database software or database libraries. 
Are these lines left over from an earlier revision of code? Are they an Easter Egg,
a reference to the SolarWinds kerfuffle from 2020, or is Jack trolling potential 
attackers by sending them down a rabbit hole? Only Jack knows.
