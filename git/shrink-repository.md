# How to Shrink a Git Repository

http://stevelorek.com/how-to-shrink-a-git-repository.html

Our main Git repository had suddenly ballooned in size. It had grown overnight to 180MB (compressed) and was taking forever to clone.

The reason was obvious; somebody, somewhere, somewhen, somehow, had committed some massive files. But we had no idea what those files where.

After a few hours of trial, error and research, I was able to nail down a process to:

-   Discover the large files
-   Clean them from the repository
-   Modify the remote (GitHub) repository so that the files are never downloaded again

This process should never be attempted unless you can guarantee that all team members can produce a fresh clone. It involves altering the history and requires anyone who is contributing to the repository to pull down the newly cleaned repository before they push anything to it.

## Deep Clone the Repository

If you don't already have a local clone of the repository in question, create one now:

`$ git clone remote-url`

Now—you may have cloned the repository, but you don't have all of the remote branches. This is imperative to ensure a proper 'deep clean'. To do this, we'll need a little Bash script:

```bash
#!/bin/bash
for branch in `git branch -a | grep remotes | grep -v HEAD | grep -v master`; do
    git branch --track ${branch##*/} $branch
done
```

Thanks to bigfish on StackOverflow for this script, which is copied verbatim.

Copy this code into a file, `chmod +x filename.sh`, and then execute it with `./filename.sh`. You will now have all of the remote branches as well (it's a shame Git doesn't provide this functionality).

## Discovering the large files

Credit is due to Antony Stubbs here - his Bash script identifies the largest files in a local Git repository, and is reproduced verbatim below:

```bash
#!/bin/bash
#set -x

# Shows you the largest objects in your repo's pack file.
# Written for osx.
#
# @see http://stubbisms.wordpress.com/2009/07/10/git-script-to-show-largest-pack-objects-and-trim-your-waist-line/
# @author Antony Stubbs

# set the internal field spereator to line break, so that we can iterate easily over the verify-pack output
IFS=$'\n';

# list all objects including their size, sort by size, take top 10
objects=`git verify-pack -v .git/objects/pack/pack-*.idx | grep -v chain | sort -k3nr | head`

echo "All sizes are in kB. The pack column is the size of the object, compressed, inside the pack file."

output="size,pack,SHA,location"
for y in $objects
do
	# extract the size in bytes
	size=$((`echo $y | cut -f 5 -d ' '`/1024))
	# extract the compressed size in bytes
	compressedSize=$((`echo $y | cut -f 6 -d ' '`/1024))
	# extract the SHA
	sha=`echo $y | cut -f 1 -d ' '`
	# find the objects location in the repository tree
	other=`git rev-list --all --objects | grep $sha`
	#lineBreak=`echo -e "\n"`
	output="${output}\n${size},${compressedSize},${other}"
done

echo -e $output | column -t -s ', '
```

Execute this script as before, and you'll see some output similar to the below:

```
All sizes are in kB. The pack column is the size of the object, compressed, inside the pack file.
size     pack    SHA                                       location
1111686  132987  a561d25105c79aa4921fb742745de0e791483afa  08-05-2012.sql
5002     392     e501b79448b9e970ab89b048b3218c2853fdfc88  foo.sql
266      249     73fa731bb90b04dcf79eeea8fdd637ba7df4c089  app/assets/images/fw/iphone.fw.png
265      43      939b31c563bd40b1ca70e4f4a9f7d67c27c936c0  doc/models_complete.svg
247      39      03514d9e84418573f26b205bae7e4e57057c036f  unprocessed_email_replies.sql
193      49      6e601c4067aaddb26991c4bd5fbddef003800e70  public/assets/jquery-ui.min-0424e108178defa1cc794ee24fc92d24.js
178      30      c014b20b6fed9f17a0b2809ac410d74f291da26e  foo.sql
158      158     15f9e56bc0865f4f303deff053e21909661a716b  app/assets/images/iphone.png
103      36      3135e15c5cec75a4c85a0636b154b83221020c97  public/assets/application-c65733a4a64a1a885b1c32694574b12a.js
99       85      c1c80bc4c09e692d5e2127e39c87ecacdb1e816f  app/assets/images/fw/lovethis_logo_sprint.fw.png
```

Yep - looks like someone has been pushing some rather unnecessary files somewhere! Including a lovely 1.1GB present in the form of a SQL dump file.

## Cleaning the files

Cleaning the file will take a while, depending on how busy your repository has been. You just need one command to begin the process:

`$ git filter-branch --tag-name-filter cat --index-filter 'git rm -r --cached --ignore-unmatch filename' --prune-empty -f -- --all`

This command is adapted from other sources—the principal addition is --tag-name-filter cat which ensures tags are rewritten as well.

After this command has finished executing, your repository should now be cleaned, with all branches and tags in tact.

## Reclaim space

While we may have rewritten the history of the repository, those files still exist in there, stealing disk space and generally making a nuisance of themselves. Let's nuke the bastards:

`$ rm -rf .git/refs/original/`
`git reflog expire --expire=now --all`
`$ git gc --prune=now`
`$ git gc --aggressive --prune=now`

Now we have a fresh, clean repository. In my case, it went from 180MB to 7MB.

## Push the cleaned repository

Now we need to push the changes back to the remote repository, so that nobody else will suffer the pain of a 180MB download.

`$ git push origin --force --all`

The --all argument pushes all your branches as well. That's why we needed to clone them at the start of the process.

Then push the newly-rewritten tags:

`$ git push origin --force --tags`

## Tell your teammates

Anyone else with a local clone of the repository will need to either use git rebase, or create a fresh clone, otherwise when they push again, those files are going to get pushed along with it and the repository will be reset to the state it was in before.
