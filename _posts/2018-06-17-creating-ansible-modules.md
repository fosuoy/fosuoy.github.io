---
layout: post
title: "Creating new Ansible modules for fun and profit!"
date: 2018-06-17
---

Working in various enterprises the requirement for idempotent deployment
processes is standard.


Being able to re-run deployments and get consistent results is a requirement for
any modern enterprise iterating towards a modern Continuous Deployment approach.

All that is required to proceed is an understanding of Python.

<div class="info">
  <h3 class="info">Quick Tip</h3>
  The rest of this article will make a lot of sense with a short understanding
  of how Ansible works.
  <br>
  The idea behind Ansible is that it copies Python scripts (or modules using
  the terminology) which run on the client hosts.
  <br>
  These modules can include within them information gathering sections to assess
  the state of the host.
  <br>
  This way the scripts can assess whether an action has to be taken (and so
  configuration is changed) or no action needs to be taken.
  <br>
  These modules then return a 'results' dictionary which includes whether an
  action has been taken (so the host has been 'changed') or not (configuration
  is 'ok').
</div>

Ok!

So for our test case, we're going to use the site:
https://jsonplaceholder.typicode.com/

Now, lets assume that if the endpoint:
https://jsonplaceholder.typicode.com/posts

Returns 100 items (which it will always do), then we want our module to run a
command specified as an option to the module.

If it returns a different number of items, we want it to do nothing.

This is a contrived example but similar issues are always present in large
enterprises.

The full module would be as below:

```
#!/usr/bin/python
# -*- coding: utf-8 -*-

__metaclass__ = type

DOCUMENTATION = '''
---
module: example_module
short_description: This is an example module
description:
 - This module will never be usefull in any setting
version_added: "2.5"
options:
  command:
    description:
     - A command to be executed in the future.

requirements:
 - python-requests
author:
- FOSUOY LTD
'''

EXAMPLES = '''
- name: Run this module! 
  example_module:
    command: hostname
  register: module_output
'''

import json
import requests

from ansible.module_utils.basic import AnsibleModule


def run_cmd(module, result, base_cmd, arguments):
    command = "%s %s" % (base_cmd, arguments)
    # check_rc = True is part of the module class
    # this will cause the module to fail if the return code from the command is
    # not 0
    rc, out, err = module.run_command(command, check_rc=True)


def get_list_length():
    '''
    Simple request, decode then return length of list
    '''
    r = requests.get('https://jsonplaceholder.typicode.com/posts')
    r_json = json.loads(r.text)
    return len(r_json)


def main():

    module = AnsibleModule(
        argument_spec=dict(
            command=dict(type='str', required=True)
        ),
        supports_check_mode=False,
    )

    at_cmd = module.get_bin_path('at', True)

    command = module.params['command']

    cmd_split = command.split(' ')
    cmd_base = module.get_bin_path(cmd_split[0], True)
    cmd_args = module.get_bin_path(' '.join(cmd_split[1:]), True)

    result = dict(
        changed=False,
        state=state,
    )

    # Find length of list from API endpoint
    list_length = get_list_length()

    if list_length == 100:
        cmd_output = run_cmd(module, result, cmd_base, cmd_args)
        result['changed'] = True
        result['list_length'] = list_length
        module.exit_json(**result)
    else:
        result['list_length'] = list_length
        module.exit_json(**result)


if __name__ == '__main__':
    main()
```

For the most part, modules like this one would be very specific to the
environment being worked in and so would not be useful to the wider community.

It is out of the scope of this article to go through deploying a module either
to the wider community or to an internal object store (e.g. Artifactory).

In the case that this is a very specific module to a certain playbook /
environment you can insert the above script into a `library/` directory in the
root of the playbook repository / directory.

After that, you can call the playbook using the name given to the file, e.g. if
you save the above as:

`library/example_module.py`

You can call the task:
```
- name: Run this module! 
  example_module:
    command: hostname
  register: module_output
```

In the above script, you can see some extra keys added to the result dictionary.

They can be called from the registered variable, e.g.:
```
- debug:
{% raw %}
    msg: "{{module_output.list_length}}"
{% endraw %}
  when:
    - module_output.changed == False
```

The above will print out the length of the list when it is not equivalent to
100.

Modules like the above will really allow your teams to fully utilize the features
of ansible in order to reduce the size of your codebase and get your systems in
a fully idempotent state when off the shelf configuration systems are not
available.
