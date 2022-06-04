---
title: Script to Clone SaltStack Formulas from GitHub
date: 2014-05-23T17:44:01+00:00
author: Iván Ádám Vári
layout: post
permalink: /script-clone-saltstack-formulas-github/
description: Python script to search all saltstack formulas available on github, then make local copies of them. If we already have then just keep our copy up to date.
keywords: python,script,github,clone,salt,saltstack,formulas
categories:
  - Python
tags:
  - python
  - salt
  - scripting
comments: true
sharing: true
footer: true
---
I am heavily into <a href="https://www.saltstack.com" target="_blank">Salt infrastructure management</a> at the moment, and wish to leverage all available (community written)
formulas. Luckily, the SaltStack group maintains a collection of excellent formulas on their <a href="https://github.com/saltstack-formulas" target="_blank">github page</a>,
and they are great source for states, ideas, best practices, etc. So I started cloning them, first the ones that I really needed. Then I realized later on, that some I may
need in the near future so why not clone all of them and ensure I have a local copy of them for my development.

The pages have been updated fairly regularly, more and more people contributing now to the project, which is great however it started to become tedious to find new formulas
and I needed an automated solution to keep up to date with the changes.

#### Script to Clone SaltStack Formulas from GitHub

I could not find anything off-the-shelf, hence I had to come up with my own solution using python.

```python
#!/usr/bin/env python
"""
    Script to find github hosted SaltStack formulas and make a up-to-date local copies of them.
"""

import urllib2
import re
import os
import subprocess


__author__ = "Ivan Vari""
__credits__ = ["Oliver Drake"]
__license__ = "GPLv3"
__version__ = None
__maintainer__ = "Ivan Vari"


URL = "https://github.com/saltstack-formulas"
PATTERN = 'a href="(/saltstack.*)"\s'
HOME = '{0}/development/saltstack-formulas'.format(os.environ['HOME'])


def main():
    """
    Main function and control flow.
    """

    # print strings with color
    # http://pythonhosted.org/ANSIColors-balises/ANSIColors.html
    colorgrn = "\033[01;32m{0}\033[00m"
    colorwht = "\033[1;37m{0}\033[00m"

    # margin for aligned printing
    width = 50

    # default loop controls
    lastpage = 1
    page = 1

    # until we reach lastpage
    while lastpage >= page:
        page_url = URL + '?page={0}'.format(page)

        connection = urllib2.Request(page_url)
        response = urllib2.urlopen(connection)

        # find all formula-references on the current page
        for match in re.finditer(PATTERN, response.read()):
            url = 'https://github.com' + match.group(1)

            # if 1st page, extract last page id and update loop controls
            if page == 1 and '?page=' in url:
                ref = re.findall('page=\d*', url)
                lastpage = int(max(ref).split('=')[1])

            # then ignore the page id completely
            if '?page=' not in url:
                # extract the formula name from the URL
                formula = url.split('/')[-1]
                repocopy = os.path.join(HOME, formula)

                # default assumes cloning
                cmd = 'git clone {0} {1}'.format(url, repocopy)
                exec_dir = '/'

                # if local copy found, change exec_dir and method to pull
                if os.path.exists(repocopy):
                    cmd = 'git pull'.format(repocopy)
                    exec_dir = repocopy

                process = subprocess.Popen(cmd.split(),
                                           stdout=subprocess.PIPE,
                                           stderr=subprocess.STDOUT,
                                           cwd=exec_dir)

                # default output definition
                result = '[OK]'
                flag = ''

                # override output definition based on git output message
                for line in process.stdout.readlines():
                    if 'Cloning' in line:
                        result = '[NEW]'
                        flag = '*'
                    elif 'Updating' in line:
                        result = '[UP]'
                        flag = '+'

                margin = width - (len(result) + len(flag))

                print '[{0}]: '.format(formula).ljust(margin, ' ') + '=>' + \
                      colorwht.format(flag) + colorgrn.format(result)
        page += 1


# boilerplate
if __name__ == "__main__":
    main()
```

This essentially parses the HTML of the front page and works out the available page numbers then fetches each page individually, filters out all formula references and runs git clone/pull over them depending whether we have local copy of it or not.
