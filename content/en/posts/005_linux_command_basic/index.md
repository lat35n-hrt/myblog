+++
title = 'Linux Command Basic'
date = '2025-04-26T16:00:00+09:00'
draft = false
+++

# 🎯 Purpose
When building a blog site with Hugo, it is essential to be able to use basic Git and Linux commands.
From a non-engineer's perspective, this table organizes the essential Linux commands for handling files and directories, based on the CRUD operations (Create, Read, Update, Delete).

# 🖥️ Linux Command Matrix (File & Directory Operations)

| Operation | File Command Example                         | Directory Command Example             | Description                                             |
|-----------|-----------------------------------------------|---------------------------------------|---------------------------------------------------------|
| Create    | `touch newfile.txt`                           | `mkdir newfolder`                     | Create a new file or directory.                         |
| Read      | `cat file.txt`<br>`less file.txt`<br>`head file.txt` | `ls foldername`                     | View the contents of a file or list the contents of a directory. |
| Update    | `nano file.txt`<br>`mv old.txt new.txt`         | `mv olddir newdir`                    | Edit the file contents or rename a file/directory.      |
| Delete    | `rm file.txt`                                 | `rm -r foldername`<br>`rmdir foldername` | Delete a file or directory.                             |

# ✍️ Notes
- `touch` only creates an empty file. You need to add content separately.
- `less` allows scrolling through the content, while `cat` displays it all at once, and `head` shows only the first few lines.
- To delete a directory: use `rm -r` if it contains files, or `rmdir` if it's empty.
- `mv` can be used for both files and directories, supporting both moving and renaming.
