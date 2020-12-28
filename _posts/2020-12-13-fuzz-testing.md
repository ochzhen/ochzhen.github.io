---
layout: post
title:  "Fuzz Testing - Overview"
tags: testing
---

Recently, I was interested in learning more about Fuzz Testing, such things as what it is, how it is used, what tools exist and how they differ between each other and, in general, if fuzz testing is really a useful thing :D

After going through different resources I decided to document my personal findings in a form of this post.

* TOC
{:toc}

## What is Fuzz Testing

**_Fuzz testing_** (fuzzing) is an automated software testing technique, so it is usually performed using a tool. This tool tries to find faults in a program by sending different inputs and observing the behavior. Inputs can be random as well as intentionally invalid and malformed.

This testing technique is not new, it emerged in 1980's, but term "fuzzing" appeared only in 1988.


## Real-World Applications

To make learning about fuzzing more interesting, I wanted to find some real-world applications of it to understand if it really works. Turns out that yes, it has many applications and extensively used in the industry. Here are some interesting cases:

- Chromium code is continuously fuzzed with 15k cores (mentioned in 2017 paper). [[2]](#useful-resources)
- For Microsoft and IE testing, Microsoft did 670 machine-years of fuzz testing during product development, generated more than 400 billion DOM manipulations (2016 blog post). [[2]](#useful-resources)
- A family of security bugs in Unix Bash shell were found in 2014. Vulnerability allowed to execute arbitrary commands. [[4]](#useful-resources)
- In 2018, use of fuzzing was demonstrated for discovery of backdoors in some x86 processors. [[5]](#useful-resources)

This list can go on, I've picked just a few examples that resonated with me.

Also, fuzzing can be used in finding regressions in REST APIs, authors call it _differential regression testing_. [[11]](#useful-resources)

Another interesting point is that fuzzing is often used by attackers to find vulnerabilities which can be exploited.


## What Can Be Tested

In my understanding, all interfaces that accept input could be fuzz tested. Moreover, one application can undergo fuzzing on different levels.

For example, if we have a web api we could test endpoints with different parameter values, how server handles malformed HTTP requests, JSON parsing, code logic itself, database, etc.

Here are some broad and quite vague categories of what can be tested that I found (this list is by no means exhaustive):

- File formats
- Parsers
- Command line tools
- GUI-based applications
- Protocols
- Libraries
- Web APIs

One point to mention, fuzzing native code, for example, applications written in C/C++ helps reveal many bugs caused by the need for developers to handle memory management. This type of problems is less relevant to managed languages, e.g. C# and Java, since they take care of memory management.


## What Faults & Problems It Looks For

In this section let us understand what a fuzzing tool looks for during execution.

Crashes and errors are the most obvious and simple to detect indicators that something went wrong in a program. For example, HTTP code 500 "Internal Server Error" clearly tells the tool that there is a significant issue.

However, many bugs do not cause system to crash immediately but are nonetheless important. To make fuzzer capable of detecting different types of issues, sanitizers can be used.

Please find below some examples of problems that can be found with the help of fuzzing. It should give a rough understanding.

- Invalid parameters handling
- Buffer overflows
- Cross-site scripting (XSS)
- SQL injections
- Race conditions and deadlocks
- Memory corruption, memory leaks
- Excessive CPU, memory consumption
- Use-after-free vulnerabilities

As we can see, fuzz testing is capable of finding many different types of bugs and vulnerabilities, and some of them are very significant.


## How Fuzzers Work

From the first glance it might appear that fuzzers just throw random inputs at the program in a hope to see it crash. Although it was a common approach in the early stages of fuzz testing, today we have more smart techniques of generating inputs that can reveal bugs in our programs.

- If we generate random data as an input, there exists an infinite number of different inputs we can try to feed into a program. However, most of them are rejected quite early since programs usually expect input to be in a certain format.

- To make the most of the resources we have for fuzz testing, it is worth generating semi-valid inputs that cover more code paths and, thus, are more likely to find issues. A common approach is to use _Fuzz Vectors_ (known-to-be-dangerous-values), this can help find issues faster by leveraging already known patterns.

- Not all vulnerabilities are exploitable, however, it is usually much easier to fix the bug than to prove that it is not exploitable.

### Generations of Fuzzers

1. **First generation** - _random fuzzers_. They generate random input and treat a program as a black box. These techniques still work today but not as well as other alternatives. We have already discussed some of the downsides of this approach. On the other hand, dumb fuzzers could be reused in more cases since they don't have specific knowledge about system under test.

2. **Second generation** - _protocol/grammar based fuzzers_. Here a tool takes some template which tells how to generate inputs. By providing a template, we constrain the set of possible things to explore.

3. **Third generation** - _instrumentation-guided fuzzing_. Fuzzers of this type incorporate feedback and learn by watching how program executes with a given input. It helps the tool to come up with new test cases based on what it has seen.

### Ways To Categorize Fuzzers

There are several ways to categorize fuzzers by its properties, any tool can belong to multiple categories mentioned below. You can read more about it on Wikipedia.

- **Generation-** or **Mutation-based**: does it need an input seed? For example, to test a program that takes an image with a mutation-based fuzzer, the tool may require a valid PNG image that it uses as a starting point to generate test cases.

- **Smart** or **Dumb**: is the tool aware of input structure or not? For example, OpenAPI specification (a.k.a. Swagger), formal grammars, file formats, network protocols.

- **Black-**, **White-**, **Gray-Box**: does it know program structure or not? Such knowledge can help achieve better code coverage.


## Advantages and Disadvantages

### Advantages

- Completely automated and quite simple, no need to come up with test cases manually.
- Fuzzing can test combinations that are not usually covered by tests written by a human.
- Good at finding unknown vulnerabilities and preventing zero day attacks.
- Fuzz testing can be used to verify static analysis outputs.

### Disadvantages

- Less effective at finding security threats that do not cause programs to crash.
- Requires significant time and resources in order to bring benefits. Running a fuzzer for 5 mins is unlikely to produce useful results.
- Usually finds relatively simple faults.


## Fuzz Testing vs Random Testing vs Monkey Testing

The short answer is that it is **_the same thing_**. However, there are some differences which may be important for you.

At first, I was a bit confused when encountered all these terms. But in general for myself as a developer I concluded that it is roughly the same. At the same time I understand that not everyone would agree with me.

Here are some points and links that might be helpful in understanding differences:

- Fuzz testing is a form of blackbox random testing
- [Answer](https://stackoverflow.com/questions/48834097/relation-between-random-testing-and-fuzz-testing){:target="_blank"} about differences between random testing and fuzz testing.
- In the Udacity [video](https://www.youtube.com/watch?v=RqrHT93KdgE){:target="_blank"} instructor tells that fuzz testing and random testing are the same.
- [Discussion](https://stackoverflow.com/questions/10241957/difference-between-fuzz-testing-and-monkey-test){:target="_blank"} about fuzz testing and monkey testing being the same thing.

Just a side note, I found [Infinite monkey theorem](https://en.wikipedia.org/wiki/Infinite_monkey_theorem){:target="_blank"} pretty fun, some people believe that it has to do with the name of monkey testing.


## Static Analysis and Fuzz Testing

Static analysis and fuzz testing are not the same:

- Static analysis does not run a program
- Static analysis makes assumptions about where problem is but not actually proves it
- Static analysis can find more bugs but needs manual verification
- Fuzz Testing explores program incrementally, reports bugs that can be reached
- Fuzz Testing requires more resources since it runs a lot of test cases against a program

Both techniques are useful and don't replace each other. Moreover, fuzz testing can be used to verify problems reported by static analysis.


## Tools

There are a number of fuzzing tools, commercial and open source. Here is a small list of what I've encountered so far: Defensics, Peach Fuzzer, Fuzzit, libFuzz, OWASP WSFuzzer, ZAProxy, OneFuzz, RESTler, etc.

Also, [GitLab acquired Peach Tech and Fuzzit](https://about.gitlab.com/press/releases/2020-06-11-gitlab-acquires-peach-tech-and-fuzzit-to-expand-devsecops-offering.html){:target="_blank"} to provide fuzz testing as a part of its CI/CD environment.

Next, we'll just talk about two tools: battle-tested AFL and quite new RESTler.

### AFL - American Fuzzy Lop

This tool is well-known among fuzzing community. You might want to take a look at the list of products where AFL managed to find bugs. For instance, Firefox, Safari, nginx, OpenSSL, MySQL are among them. More examples [here](https://lcamtuf.coredump.cx/afl/){:target="_blank"}.

By the way, it is named after a [cute rabbit breed](https://en.wikipedia.org/wiki/American_Fuzzy_Lop){:target="_blank"}.

- Mutation-based, instrumentation-guided fuzzer
- Takes user-supplied test cases as input
- Generates a small corpus of interesting test cases as a result of fuzzing
- Offers white- and black-box instrumentation based on source code availability

For detailed information please visit [AFL GitHub](https://github.com/google/AFL){:target="_blank"}.

### RESTler

RESTler is a stateful REST API fuzzing tool. This is quite new project developed by Microsoft Research.

- Takes OpenAPI specification (Swagger) as an input
- Infers producer-consumer dependencies, e.g. response from A is an argument for B.
- Learns from service responses, e.g. request C is refused after executing A and B.
- Authors showed how it can be used in differential regression testing.

Despite being relatively new, it has already found more than a hundred issues in Azure's and GitLabs REST APIs.

More information about RESTler could be found in papers mentioned in [RESTler GitHub](https://github.com/microsoft/restler-fuzzer){:target="_blank"}.


## Summary

- Fuzz testing is a useful software testing technique that helps find unknown vulnerabilities in an automated fashion without the need to create test cases manually.
- Fuzzing is just one piece of software testing and should be used along with other testing techniques.
- There are 3 generations of fuzzers and the field is still evolving.
- Fuzzing allows to test software on different levels, it is especially useful for native code where developer has to handle memory management.
- There are a number of commercial and open source fuzzing tools that are proven to be effective.


## Useful Resources

1. [https://en.wikipedia.org/wiki/Fuzzing](https://en.wikipedia.org/wiki/Fuzzing){:target="_blank"}
2. [https://browser-security.x41-dsec.de/X41-Browser-Security-White-Paper.pdf](https://browser-security.x41-dsec.de/X41-Browser-Security-White-Paper.pdf){:target="_blank"}
3. [https://owasp.org/www-community/Fuzzing](https://owasp.org/www-community/Fuzzing){:target="_blank"}
4. [https://www.nytimes.com/2014/09/26/technology/security-experts-expect-shellshock-software-bug-to-be-significant.html](https://www.nytimes.com/2014/09/26/technology/security-experts-expect-shellshock-software-bug-to-be-significant.html){:target="_blank"}
5. [https://www.blackhat.com/us-18/briefings/schedule/#god-mode-unlocked---hardware-backdoors-in-x86-cpus-10194](https://www.blackhat.com/us-18/briefings/schedule/#god-mode-unlocked---hardware-backdoors-in-x86-cpus-10194){:target="_blank"}
6. [https://www.techrepublic.com/article/fuzzing-fuzz-testing-tutorial-what-it-is-and-how-can-it-improve-application-security/](https://www.techrepublic.com/article/fuzzing-fuzz-testing-tutorial-what-it-is-and-how-can-it-improve-application-security/){:target="_blank"}
7. [https://builtin.com/software-engineering-perspectives/fuzz-testing](https://builtin.com/software-engineering-perspectives/fuzz-testing){:target="_blank"}
8. [https://owasp.org/www-project-web-security-testing-guide/v41/6-Appendix/C-Fuzz_Vectors](https://owasp.org/www-project-web-security-testing-guide/v41/6-Appendix/C-Fuzz_Vectors){:target="_blank"}
9. [https://github.com/google/AFL](https://github.com/google/AFL){:target="_blank"}
10. [https://github.com/microsoft/restler-fuzzer](https://github.com/microsoft/restler-fuzzer){:target="_blank"}
11. [https://patricegodefroid.github.io/public_psfiles/issta2020.pdf](https://patricegodefroid.github.io/public_psfiles/issta2020.pdf){:target="_blank"}
12. [https://en.wikipedia.org/wiki/Differential_testing](https://en.wikipedia.org/wiki/Differential_testing){:target="_blank"}
