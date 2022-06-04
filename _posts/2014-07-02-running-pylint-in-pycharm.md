---
title: Running Pylint in PyCharm
date: 2014-07-02T22:41:28+00:00
author: Iván Ádám Vári
layout: post
permalink: /running-pylint-in-pycharm/
description: I liked the seamless Pylint integration in Eclipse/Pydev and after I switched permanently to PyCharm, I was curious how I could run pylint in pycharm.
keywords: pylint,pychram,plugin
categories:
  - Python
tags:
  - ide
  - python
comments: true
sharing: true
footer: true
---
I really liked the <a href="http://www.pylint.org" target="_blank">Pylint</a>integration in Eclipse/Pydev but I have switched to
<a href="http://www.jetbrains.com/pycharm/" target="_blank">PyCharm</a> since JetBrains released CE edition. Pycharm supports
<a href="http://legacy.python.org/dev/peps/pep-0008/" target="_blank">PEP8</a>auditing "out of the box", but I found out lately, that it is a little "loose" on style
compared to pylint. Running pylint in pycharm didn't seem to be supported in any ways so I became curious about how I could add this functionality to my favourite IDE.

After some searching, I realised that there is not much out there about this topic. I could not accept it and went after the challenge...

## Running Pylint in PyCharm

As it turned out, it is much simpler than I initially thought. First install pylint, it's very easy, there are various ways of doing it depending on operating system, your
preference, etc. so the installation is out of the scope of this guide.

#### Create an external tool:
![Config](/assets/img/2014-07/968C2439-5AFA-4B48-8094-8895F3C8A3C5-1404279043311.png)

My pylint config is fairly simple, nearly stock standard so I simply added my customisation to the "parameters" line, but for more complex setups, I strongly recommend setting
up `.pylintrc` in your HOME then adding your config file to the parameter line as an argument. This basically pipes your code through pylint and displays the results in the "run"
console.

#### Configure output filter:

![Filter](/assets/img/2014-07/B380E5C8-39A2-4C43-BFE2-E8796EE4F5A4.png)

These will create an XML config that looks like this on OSX:


```bash
$ less Library/Preferences/PyCharm4.0/tools/CodeCompliance.xml
```

```xml
<toolSet name="CodeCompliance">
  <tool name="Pylint" description="External module to integrate pylint compliance checks into PyCharm" showInMainMenu="true" showInEditor="true" showInProject="true" showInSearchPopup="true" disabled="false" useConsole="true" showConsoleOnStdOut="false" showConsoleOnStdErr="false" synchronizeAfterRun="true">
    <exec>
      <option name="COMMAND" value="pylint"/>
      <option name="PARAMETERS" value="--disable=W0106,W0212 --max-line-length=120 --msg-template=&quot;$FileDir$/{path}:{line}: [{msg_id}({symbol}), {obj}] {msg}&quot; $FilePath$"/>
      <option name="WORKING_DIRECTORY" value="$FileDir$"/>
    </exec>
    <filter>
      <option name="NAME" value="ErrorMessages"/>
      <option name="DESCRIPTION" value="Creates jump links to non-compliant lines"/>
      <option name="REGEXP" value="$FILE_PATH$\:$LINE$\:"/>
    </filter>
  </tool>
</toolSet>
```

As the description says, this will essentially give you quick _jump links_ to the parts of your code where the non-compliant code is.

![Console](/assets/img/2014-07/A6605616-D1FD-4FFB-B67C-C87D80B331D6.png)

>Clicking on any of the blue lines will simply navigate you to the line where the issue is, and adding _keymap_ to this tool can improve your workflow too.
{: .prompt-tip }

