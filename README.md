# Cybersecurity Portfolio

A record of my hands-on cybersecurity training: labs, exercises, and writeups documented as I work through them. Each writeup shows what I did, the commands and output behind it, why I made each choice, and what I learned, including the dead ends.

I'm building toward penetration testing and security analysis roles, and this repository is where I show my working.

## Credentials

- **CompTIA Security+**
- **Google Cybersecurity Certificate**

## What's Here

Everything I've documented from my learning journey: training platform modules, lab machines, challenges, and exercises across offensive and defensive security. Work is organised by platform, then by course, path or category, then by module, lab or challenge. Each module, lab or challenge has its own folder holding its writeups and its own `img/` folder for screenshots:

```
<Platform>/
└── <Course, Path or Category>/
    ├── <Module>/
    │   ├── 1. <Section>.md
    │   ├── 2. <Section>.md
    │   └── img/
    └── <Lab or Challenge>/
        ├── <Writeup>.md
        └── img/
```

Browse the folders to see what's covered. The repository grows as I complete new material.

> **Work in progress:** I'm in the middle of migrating my notes from a personal Obsidian vault, redacting and cleaning each writeup as it moves over. Most of my completed work isn't here yet, and new modules will keep appearing.

## How Writeups Are Written

Each exercise follows a consistent pattern, so any writeup is quick to scan:

1. **Question and context**: what was asked and what the target looked like.
2. **Approach**: how I analysed the problem and why I chose the tools and options I did.
3. **Commands and output**: the full command with real tool output, trimmed with `—SNIP—` where it adds nothing.
4. **Breakdown**: what each flag or step does.
5. **Result and takeaways**: what the output means and what went wrong along the way.

Screenshots have descriptive filenames and alt text.

## Redactions

All work here was done on authorised, intentionally vulnerable lab environments. Some content is redacted and shown as `—REDACTED—`:

- **Flags and answers.** Training platforms such as Hack The Box ask that solutions to their course material not be published. The method and the reasoning are what I'm sharing.
- **Personal identifiers.** My usernames and local file paths are removed from terminal output and screenshots.

Lab IP addresses and sample data come from temporary training environments and are not sensitive.

## Disclaimer

Everything in this repository was performed in legal training environments for educational purposes. Don't use these techniques against systems you don't own or have explicit permission to test.
