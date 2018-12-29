# challenge

> This flag is protected by a password stored in a highly sohpisticated chain of hashes. Can you capture it nevertheless? We are certain the password consists of lowercase alphanumerical characters only.
> 
> nc 35.207.158.95 1337
> 
> Source
> 
> Difficulty estimate: Easy

Along come a couple of files

```
pseyfert@robusta:/tmp/ultrasecret > tree
.
├── Cargo.lock
├── Cargo.toml
└── src
    └── main.rs
```

The only one I found interesting is `main.rs`:

```
extern crate crypto;
  
use std::io;
use std::io::BufRead;
use std::process::exit;
use std::io::BufReader;
use std::io::Read;
use std::fs::File;
use std::path::Path;
use std::env;

use crypto::digest::Digest;
use crypto::sha2::Sha256;

fn main() {
    let mut password = String::new();
    let mut flag = String::new();
    let mut i = 0;
    let stdin = io::stdin();
    let hashes: Vec<String> = BufReader::new(File::open(Path::new("hashes.txt")).unwrap()).lines().map(|x| x.unwrap()).collect();
    BufReader::new(File::open(Path::new("flag.txt")).unwrap()).read_to_string(&mut flag).unwrap();

    println!("Please enter the very secret password:");
    stdin.lock().read_line(&mut password).unwrap();
    let password = &password[0..32];
    for c in password.chars() {
        let hash =  hash(c);
        if hash != hashes[i] { 
            exit(1);
        } 
        i += 1;
    }   
    println!("{}", &flag)
}            

fn hash(c: char) -> String {
    let mut hash = String::new(); 
    hash.push(c);
    for _ in 0..9999 {
        let mut sha = Sha256::new();
        sha.input_str(&hash);
        hash = sha.result_str();
    }
    hash
}
```

Despite not knowing rust, I can tell:

 - The password is 32 characters
 - when it's entered each character gets hashed, and the hash is compared to a hash stored in `hashes.txt` (there should be 32 hashes in it, the first for the first character, and so on)
 - The hash function uses sha256 and the character is hashed, the hash is hashed again, and again and again.

# how the solution dawns

As a blind stab I wanted to know what the hash of a hash of a hash ... might
look like and when typing it as a simple pipe-sequence on the shell wondered
"How many characters can the shell digest … how long will that hashing take …
what would I do with the hash tab… … wait … timing!" The check loop aborts once
the first character is wrong. i.e. if I just enter a long password many times
with only different first letters, each password attempt will only call one
long hash computation, except if I hit the right first letter - then two hashes
will be computed (the second character gets checked and is presumably wrong).

So, I ended up with a first quick loop:

```sh
for c in {0..9} {a..z}; do echo $c ; time echo "${c}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | nc 35.207.158.95 1337 ; done
```

The result is:
```
0
Please enter the very secret password:
echo "${c}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  0,00s user 0,00s system 52% cpu 0,001 total
nc 35.207.158.95 1337  0,00s user 0,00s system 0% cpu 0,960 total
1
Please enter the very secret password:
echo "${c}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  0,00s user 0,00s system 52% cpu 0,002 total
nc 35.207.158.95 1337  0,01s user 0,00s system 0% cpu 1,652 total
2
Please enter the very secret password:
echo "${c}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  0,00s user 0,00s system 53% cpu 0,002 total
nc 35.207.158.95 1337  0,00s user 0,01s system 1% cpu 0,960 total
3
Please enter the very secret password:
echo "${c}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"  0,00s user 0,00s system 57% cpu 0,002 total
nc 35.207.158.95 1337  0,01s user 0,01s system 1% cpu 0,926 total
...
```

Yeah, the first symbol is a `1`.

## Tedious work

Now one just needs to iterate that over all characters, the loop taking longer
and longer every time.

I did not automize that and noticed another player getting trapped at one
point. Manual running had these advantages (assuming you do something useful in
parallel, like watching youtube):

 - I aborted the first loops, because many characters were digits, thus saving
   time over running all characters. In fact, I could reduce the loop long
   before reaching slow computations because I suspected the password is in hex.
 - There was some human judgeing how reliable the timing was. In the
   for-writeup quote above, the timings were way better than in the night when
   I ran. On long passwords, the wrong-times (that should all be similar)
   fluctuated by 300ms on feeling-average and sometimes up to 700ms. The
   difference between N and N+1 hash computations is only around 700ms, so there
   were a few characters where a blind max-time lead to the wrong character. As a
   human I saw when I picked the wrong character that the average time on the next
   character didn't increase wrt the previous loop. The worst was that at some
   point the server load seemingly dropped and there was a step down in the
   average timings half way through my loop.

Though, parallel checking of letters from a multithreaded pool is even with
script-writing time taken into account faster …

The loop for the last letter already shows the flag somewhere in the middle of
the printout. Here the quoted:

```
pseyfert@robusta:/tmp/k > echo 10e004c2e186b4d280fad7f36e779ed4 | nc 35.207.158.95 1337
Please enter the very secret password:
35C3_timing_attacks_are_fun!_:)
```

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/80x15.png) This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)]
