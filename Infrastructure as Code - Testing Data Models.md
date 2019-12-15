# Infrastructure as Code - Testing Data Models

As many organisations begin the journey into DevOps and start defining their infrastructure as code, one of the issues they begin to face is data model accuracy.

Two of the most popular automation tools today are ansible and ansible tower. These tools provide frameworks to abstract the desired state of infrastructure, whether it be compute, storage, cloud, networks etc and schedule or trigger playbook runs based on that abstraction. This abstraction makes use of predefined or custom modules which in turn are feed by user defined data models. This article will cover testing and validating the data within these user defined data models.

## Use-case problem
Take this simple data model;

```
# roles/switch/vars/main.yml
---
site_vlans:
  - { vlan_name: test, vlan_id: 276, tags: ['pe01'] }
  - { vlan_name: test, vlan_id: 277, tags: ['test_mgmt'] }
  - { vlan_name: test, vlan_id: 278, tags: ['fw', 'cpe'] }
  - { vlan_name: test, vlan_id: 280, tags: ['potato'] }
```

What this defines is a series of vlans for a site. Each vlan has a vlan_name, vlan_id and a list of tags. The keys are all used by the ansible template module the generates the ASCII configuration files.

```
# roles/switch/tasks/main.yml
---
- name: Leaf configs - VLANs
  template:
    src: vlans.j2
    dest: /tmp/{{inventory_hostname}}/vlans.conf
  check_mode: no


# roles/switch/templates/vlans.j2
{% for vlan in site_vlans %}
!
vlan {{vlan.vlan_id}}
   name {{vlan.vlan_name}}_{{vlan.vlan_id}}
{% endfor %}
```

While for this file this would not be an issue in it current state, we need to think about what could happen should someone accidentally miss-type one of the keys.

```
# roles/switch/vars/main.yml
---
site_vlans:
  - { vlan_name: test-1, vlan_id: 276, tags: ['pe01'] }
  - { vlan_name: test-2, vlan_id: 277, tags: ['test_mgmt'] }
  - { vlan_name: test-3, vlroles/switch/vars/main.ymlan_id: 278, tags: ['fw', 'cpe'] }
  - { vlan_name: test-4, vlan_id: 280, tags: ['potato'] }
  - { vlan_namee: test-5, vlan_id: 281, tags: ['lima'] }
```

The last line has an error, the key vlan_name is mis-typed. If we were to run the associated task which calls the jinja2 template the play would fail with an exception stating that key 'vlan_name' is not defined.

One way around this is to have business policies preventing merges of code changes into the master branch without first going through review via a pull request. While this example is quite simple, as data models grow, the number of models expands or the number of merge requests increase the likelihood of both the reviewer and submitter missing this type of error goes up. When errors are merged, scheduled or triggered automation jobs that rely on an accurate master branch are prone to failure, resulting in longer provisioning lead-times and rework and possibly infrastructure outages.

## Use-case Solution - Intro
One option I like it [pre-commit](https://pre-commit.com/). This is a framework written in python that you install to manage the commit process. Upon a commit being executed pre-commit will, compile a list of the changed files and pass that list into the configured hooks. Each test is either run or skipped (based on filters), and when run it returns an exit code. Zero for a passed test and one or more for failed test. Any failed test results in the commit being denied, forcing errors to be resolve prior to commit.

Such a tool is used to complement the business policies around submission of code and merge requests. Reviewers and submitter's get a level of confidence that necessary tests have been successfully carried out.

To test the above data model we first need to install pre-commit;

```
$ pip3 install pre-commit --user


Collecting pre-commit
  Using cached https://files.pythonhosted.org/packages/89/97/fe584f47dc43332ac254ed3940d2a3401877be73e3150a557641c9f812a6/pre_commit-1.20.0-py2.py3-none-any.whl
Collecting aspy.yaml
  Using cached https://files.pythonhosted.org/packages/99/ce/78be097b00817ccf02deaf481eb7a603eecee6fa216e82fa7848cd265449/aspy.yaml-1.3.0-py2.py3-none-any.whl
Collecting cfgv>=2.0.0
  Using cached https://files.pythonhosted.org/packages/6e/ff/2e6bcaff26058200717c469a0910da96c89bb00e9cc31b68aa0bfc9b1b0d/cfgv-2.0.1-py2.py3-none-any.whl
Collecting toml
  Using cached https://files.pythonhosted.org/packages/a2/12/ced7105d2de62fa7c8fb5fce92cc4ce66b57c95fb875e9318dba7f8c5db0/toml-0.10.0-py2.py3-none-any.whl
Collecting virtualenv>=15.2
  Using cached https://files.pythonhosted.org/packages/62/77/6a86ef945ad39aae34aed4cc1ae4a2f941b9870917a974ed7c5b6f137188/virtualenv-16.7.8-py2.py3-none-any.whl
Collecting importlib-metadata; python_version < "3.8"
  Using cached https://files.pythonhosted.org/packages/f6/d2/40b3fa882147719744e6aa50ac39cf7a22a913cbcba86a0371176c425a3b/importlib_metadata-0.23-py2.py3-none-any.whl
Processing ./.cache/pip/wheels/7b/6c/23/eb26369b77904c8963fae9e64338b0f0b948b4d59710760834/nodeenv-1.3.3-cp36-none-any.whl
Collecting importlib-resources; python_version < "3.7"
  Using cached https://files.pythonhosted.org/packages/2f/f7/b4aa02cdd3ee7ebba375969d77c00826aa15c5db84247d23c89522dccbfa/importlib_resources-1.0.2-py2.py3-none-any.whl
Collecting six
  Using cached https://files.pythonhosted.org/packages/65/26/32b8464df2a97e6dd1b656ed26b2c194606c16fe163c695a992b36c11cdf/six-1.13.0-py2.py3-none-any.whl
Collecting identify>=1.0.0
  Using cached https://files.pythonhosted.org/packages/87/e4/66e3c82550017d3ee03c9f216e0c3dbf1c8c580c567777537adce8823597/identify-1.4.7-py2.py3-none-any.whl
Processing ./.cache/pip/wheels/d9/45/dd/65f0b38450c47cf7e5312883deb97d065e030c5cca0a365030/PyYAML-5.1.2-cp36-cp36m-linux_x86_64.whl
Collecting zipp>=0.5
  Using cached https://files.pythonhosted.org/packages/74/3d/1ee25a26411ba0401b43c6376d2316a71addcc72ef8690b101b4ea56d76a/zipp-0.6.0-py2.py3-none-any.whl
Collecting more-itertools
  Using cached https://files.pythonhosted.org/packages/45/dc/3241eef99eb45f1def35cf93af35d1cf9ef4c0991792583b8f33ea41b092/more_itertools-7.2.0-py3-none-any.whl
Installing collected packages: pyyaml, aspy.yaml, six, cfgv, toml, virtualenv, more-itertools, zipp, importlib-metadata, nodeenv, importlib-resources, identify, pre-commit
Successfully installed aspy.yaml-1.3.0 cfgv-2.0.1 identify-1.4.7 importlib-metadata-0.23 importlib-resources-1.0.2 more-itertools-7.2.0 nodeenv-1.3.3 pre-commit-1.20.0 pyyaml-5.1.2 six-1.13.0 toml-0.10.0 virtualenv-16.7.8 zipp-0.6.0
```

Next it is necessary to create a file called ./pre-commit-config.yaml in the root of our repository.

```
# ./pre-commit-config.yaml

---
default_stages: [commit, push, manual]


repos:
  - repo: https://gitlab.com/johnsondnz/pre-commit-example.git
    rev: 1.0.0
    hooks:
      - id: vlan-keys
      - id: vlan-duplicates
      - id: ansible-lint
```

In the above configuration we pull the hooks in from a second repository that contain the tests that need to be carried out. Pre-commit will send the list containing the relative path to each changed file into the vlan-duplicates, vlan-keys, and ansible-lint hooks when ever the developer or engineer executes a `git commit`.

Before we can use this there is one additional step. Git has to have a file called .git/hooks/pre-commit in order to make use of the hooks. Pre-commit handles this for us with one easy to use command.

```
$ pre-commit install
pre-commit installed at .git/hooks/pre-commit
```

Pre-commit is ready to go. Simply make some changes and the tests will be carried out automatically. If, for any reason the tests fail the commit is denied. If to any reason we need to bypass this, the `--no-verify` flag can be added to a commit, no tests are carried out though, so beware!!

## Use-Case Solution - The Guts of it
In our example we are concerned about bad keys. `vlan_name` is missing and a key `vlan_namee` is in its place. To detect this a test exists. This test called vlan-keys in our pre-commit-config.yaml file checks for a predefinned list of keys and that they exist.

Defined the allowed keys for a vlan as well as the absolute minimum keys required for successful provisioning,

```
ALLOWED_KEYS = ['vlan_name', 'vlan_id', 'tags']
MINIMAL_KEYS = ALLOWED_KEYS[:3]
```

To do this it first loads the file into a python dictionary, so we can iterate over these to validate the keys we are expecting, if there is an issue opening the file we need to handle that too.

```
try:
    with open(filename) as f:
        site_vlans = yaml.load(f, Loader=yaml.FullLoader)["site_vlans"]
```

From here the vlans now exist as a variable called site_vlans. Next we iterate over the vlans and the keys within to verify if that key is in the minimum list, if it is not then a message is printed to stdout and error set to True.

```
    for vlan in site_vlans:
        if not set(MINIMAL_KEYS).issubset(vlan.keys()):
            print(f"File: {filename} - Vlan: {vlan['vlan_id']} - Missing keys {list(set(MINIMAL_KEYS) - set(vlan.keys()))}")
            error = True    
```

This check will verify that, vlan_name, vlan_id and tag keys all exist for each vlan. If a vlan is missing any of these keys we have an error and a message is printed.

Next we validate that each key is in the allowed list. It is possible that additional systems on top of our switches need vlans as well, and they may need or allow for additional data. This check verifies that all keys for a vlan and in the allowed list. If a vlan has an unknown key we have an error and a message is printed.

```
        for key in vlan:
            if key not in ALLOWED_KEYS:
                print(f"Filenmae: {filename} - Vlan: {vlan['vlan_id']} - Invalid keys: {list(set(vlan.keys() - set(ALLOWED_KEYS)))}")
                error = True


except (IOError, Exception):
    print(f"Something went wrong opening the file: {filename}")
    error = True
    pass
```

In our case we'd expect to see the following two errors during run-time resulting in commit failures until the issue is resolved.

```
File: roles/switch/vars/main.yml - Vlan: 281 - Missing Keys ['vlan_name']
File: roles/switch/vars/main.yml - Vlan: 281 - Invalid Keys ['vlan_namee']
```

## Conclusion
While it is possible with business policies to control code merges into master, errors will invariably occur. Testing is designed to complement business policies and strengthen the DevOps culture. It's important that we shorten the time to detect as much as possible. As part of the three-ways of DevOps, we should ensure no faults pass downstream and that feedback loops are created and amplified. By executing tests at the time of commit many issues can be caught before they enter the review process or result in automation failures.

It is critical that tests continue to be developed as we discover issue in our code and its execution to prevent future occurrences.

Ideally in our example we would test each key's data as well. This would ensure there are no spaces in the vlan_name, that the vlan_id is an integer between 1 and 4094 and that the tags are relevant to the rest of the playbook, i.e trunks exist with the tags association for vlans to be mapped to in our logic.

Pre-commit comes with great documentation and core built in hooks for you to use today. Building test cases is super easy and fast to implement. I encourage you to play and let me know what you come up with.

Example pre-commit hooks
My hook examples can be [found here on gitlab](https://gitlab.com/johnsondnz/pre-commit-example) and a [demo playbook](https://gitlab.com/depereo/cloudbuilders-demo) using Arista container EOS here courtesy of [@depereo](https://gitlab.com/depereo) for anyone wanting to play with any of these examples.
