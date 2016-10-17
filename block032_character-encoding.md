# Character encoding



*under development ... not clear where this is going*

## Character encoding: where it fits in

Main page about character data: [Character vectors](block028_character-data.html)

## Resources

  * [Strings subsection of data import chapter](http://r4ds.had.co.nz/data-import.html#readr-strings) in R for Data Science
  * Screeds on the Minimum Everyone Needs to Know about encoding
    - [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html)
    - [What Every Programmer Absolutely, Positively Needs To Know About Encodings And Character Sets To Work With Text](http://kunststube.net/encoding/)
  * [Guide to fixing encoding problems in Ruby](http://www.justinweiss.com/articles/3-steps-to-fix-encoding-problems-in-ruby/) *parking here temporariliy ... looks useful but, obviously, it's about Ruby not R*

## What is an encoding?

Translating this post literally from Ruby to R: [Guide to fixing encoding problems in Ruby](http://www.justinweiss.com/articles/3-steps-to-fix-encoding-problems-in-ruby/)
 
Look at the string `"hello!"` in bytes.
 
```
irb(main):001:0> "hello!".bytes
=> [104, 101, 108, 108, 111, 33]
```
 
The base function `charToRaw()` "converts a length-one character string to raw bytes. It does so without taking into account any declared encoding". It displays bytes in hexadecimal. Use `as.integer()` to convert to decimal.


```r
charToRaw("hello!")
#> [1] 68 65 6c 6c 6f 21
as.integer(charToRaw("hello!"))
#> [1] 104 101 108 108 111  33
```

Use a character less common in English:

```
irb(main):002:0> "hellṏ!".bytes
=> [104, 101, 108, 108, 225, 185, 143, 33]
```


```r
charToRaw("hellṏ!")
#> [1] 68 65 6c 6c e1 b9 8f 21
as.integer(charToRaw("hellṏ!"))
#> [1] 104 101 108 108 225 185 143  33
```

Now we see that the `"ṏ"` is represented by more than one byte. Three in fact: [225, 185, 143]. The encoding of a string defines this relationship.

Take a look at what a single set of bytes looks like when you try different encodings.

First, an ISO-8859-1 string with a special character.

```
irb(main):003:0> str = "hellÔ!".encode("ISO-8859-1"); str.encode("UTF-8")
=> "hellÔ!"

irb(main):004:0> str.bytes
=> [104, 101, 108, 108, 212, 33]
```


```r
string <- "hellÔ!"
Encoding(string)
#> [1] "unknown"
string_latin <- iconv(string, from = "UTF-8", to = "Latin1")
string_latin
#> [1] "hell\xd4!"
charToRaw(string_latin)
#> [1] 68 65 6c 6c d4 21
as.integer(charToRaw(string_latin))
#> [1] 104 101 108 108 212  33
```

What would that string look like interpreted as ISO-8859-5 instead?
```
irb(main):005:0> str.force_encoding("ISO-8859-5"); str.encode("UTF-8")
=> "hellд!"
```

```r
iconv(string_latin, from = "ISO-8859-5", to = "UTF-8")
#> [1] "hellд!"
```

And not all strings can be represented in all encodings:
```
irb(main):006:0> "hi∑".encode("Windows-1252")
Encoding::UndefinedConversionError: U+2211 to WINDOWS-1252 in conversion from UTF-8 to WINDOWS-1252
 from (irb):61:in `encode'
 from (irb):61
 from /usr/local/bin/irb:11:in `<main>'
 ```

```r
(string <- "hi∑")
#> [1] "hi∑"
Encoding(string)
#> [1] "unknown"
as.integer(charToRaw(string))
#> [1] 104 105 226 136 145
(string_windows <- iconv(string, from = "UTF-8", to = "Windows-1252"))
#> [1] NA
```

In Ruby, apparently that is an error. In R, we just get `NA`. Alternatively, and somewhat like Ruby, you can specify a substitution for non-convertible bytes.


```r
(string_windows <- iconv(string, from = "UTF-8", to = "Windows-1252", sub = "?"))
#> [1] "hi???"
```

In the Ruby post, we've seen 3 string functions so far.

  * `encode` translates a string to another encoding. We've used `iconv(x, from = "UTF-8", to = <DIFFERENT_ENCODING>)` here.
  * `bytes` shows the bytes that make up a string. We've used `charToRaw()`, which returns hexadecimal in R. For the sake of comparison to the Ruby post, I've converted to decimal with `as.integer()`.
  * `force_encoding` shows what the input bytes would look like if interpreted by a different encoding. We've used `iconv(x, from = <DIFFERENT_ENCODING>, to = "UTF-8")`.

## A three-step process for fixing encoding bugs

### Discover which encoding your string is actually in.

Shhh. Secret: this is encoded as Windows-1252. `\x99` should be the trademark symbol ™.

```ruby
irb(main):078:0> "hi\x99!".encoding
=> #<Encoding:UTF-8>
```

This is not encoded as UTF-8, despite what Ruby thinks. R admits it doesn't know and `stringi`'s guess is not good.


```r
string <- "hi\x99!"
string
#> [1] "hi\x99!"
Encoding(string)
#> [1] "unknown"
stringi::stri_enc_detect(string)
#> [[1]]
#> [[1]]$Encoding
#> [1] "UTF-16BE" "UTF-16LE" "EUC-JP"   "EUC-KR"  
#> 
#> [[1]]$Language
#> [1] ""   ""   "ja" "ko"
#> 
#> [[1]]$Confidence
#> [1] 0.1 0.1 0.1 0.1
```

Advice given in post is to sleuth it out based on where the data came from or to look at encoding tables.

### Decide which encoding you want the string to be

That's easy. UTF-8. Done.

### Re-encode your string

```
irb(main):088:0> "hi\x99!".encode("UTF-8", "Windows-1252")
=> "hi™!"
```


```r
string_windows <- "hi\x99!"
Encoding(string)
#> [1] "unknown"
string_utf8 <- iconv(string_windows, from = "Windows-1252", to = "UTF-8")
Encoding(string_utf8)
#> [1] "UTF-8"
string_utf8
#> [1] "hi™!"
```