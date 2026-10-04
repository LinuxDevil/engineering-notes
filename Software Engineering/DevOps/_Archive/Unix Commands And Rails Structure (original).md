---
title: "Unix Commands And Rails Structure (original)"
---
Common Unix Commands List:


| Command | Example | Description |
| --- | --- | --- |
| `cd` | `$ cd ..` | cd one directory up |
| `cd` | `$ cd ~ or just $ cd` | cd to home directory |
| `cd` | `$ cd ~/environment/` | cd to path including home dir |
| `cd <dirname>` | `$ cd environment/` | change directory |
| `cat <file>` | `$ cat ~/.ssh/id_rsa.pub` | concatenate and display file contents |
| `cp <source> <target>` | `$ cp foo bar` | copy file |
| `ls` | `$ ls -l` | list contents |
| `mkdir <dirname>` | `$ mkdir environment` | make directory |
| `mv <source> <target>` | `$ mv foo bar` | move file (rename) |
| `rmdir <directory>` | `$ rmdir environment/` | remove empty directory |
| `rm <file>` | `$ rm foo` | remove file |
| `rm -rf <directory>` | `$ rm -rf tmp/` | remove nonempty directory |


File Structure for Rails application using the rails new command


| File/Directory | Purpose |
| --- | --- |
| `README.md` | A brief description of the application |
| `config.ru` | A configuration file for Rack middleware |
| `Gemfile.lock` | A list of gems used to ensure that all copies of the app use the same gem |
| `bin/rails` | A program for generating code, opening console sessions, or starting a |
| `app/assets` | Application assets such as Cascading Style Sheets (CSS) and images |
| `config/` | Application configuration |
| `log/` | Application log files |
| `test/` | Application tests |
| `bin/` | Binary executable files |
| `app/` | Core application (app) code, including models, views, controllers, and |
| `public/` | Data accessible to the public (e.g., via web browsers), such as error pages |
| `db/` | Database files |
| `doc/` | Documentation for the application |
| `Gemfile` | Gem requirements for this app |
| `lib/` | Library modules |
| `.gitignore` | Patterns for files that should be ignored by Git |
| `tmp/` | Temporary files |

