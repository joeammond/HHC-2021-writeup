# Introduction

This year's [Holiday Hack Challenge](https://2021.kringlecon.com)
was an interesting mix of challenges, including web app hacking,
SQL injection, and VHDL programming. One challenge in particular
involved abusing a field in a web form for a [Server-Side Request
Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
**(SSRF)** that can be used to steal an access key to Amazon Web Services. My
writeup will focus how the SSRF in the application can also be abused
to download the application source, enumerate the container running
the application, and reverse-engineer it to run a local copy. I'll
also detail ways in which an attacker can leverage a [Local File
Inclusion](https://cobalt.io/blog/a-pentesters-guide-to-file-inclusion) **(LFI)**
vulnerability to read more than just files: for example, running processes
or open network connections can be determined as well.

My writeup doesn't include any of the other challenges. For the remaining
ones I don't cover, I recommend reading [@CraHan's](https://twitter.com/crahan)
excellent writeup, available [here](https://n00.be/HolidayHackChallenge2021/).
