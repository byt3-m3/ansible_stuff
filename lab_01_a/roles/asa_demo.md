# Advanced Ansible ASA Playbook Demo

**Important** *Data Examples have been cut down for reading purposes. Data modules used for  this demo will be available on github* **Important** 

Today i will be illustrating the use of Ansible in a NetDevOps environment working with Cisco ASA firewalls. My main goal for this demo is to show
how jinia2 templates can be utilized with host_var and group_var variables to create a more dynamic means of configuration management.  

First I need to create a model that represents my Cisco ASA firewall. I will use YAML for this demonstration but an .ini hosts file can be 
used as well if you are more comfortable with that style of syntax. Lets address the need of this playbook, without a goal you can find
yourself going off on the deep end pretty easily so lets address my problem.

#### Requirements:
 - need a way to push ACLs to my device
 - ability to maintain the current configuration state of my firewall.
 - I need a tool that error checks for pre-existing object-groups and ACLs
 - standardization of ACL rules
 - need a text file generated containing ACL rules for auditors 
 
 Now my end goal is to create a rule set that looks like this 
 
 ```text
 access-list INSIDE-OUT remark ###############################################
access-list INSIDE-OUT remark #     Rule for Rules for Internal Networks
access-list INSIDE-OUT remark ###############################################
access-list INSIDE-OUT extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group DMZ200_NETS log 
access-list INSIDE-OUT extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group EXTERNAL_NETS log 
access-list INSIDE-OUT extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group ANY_NET log 
access-list INSIDE-OUT extended permit object-group ICMP_PPSM object-group INSIDE_NETS object-group ANY_NET log 
access-list INSIDE-OUT extended permit object-group WINDOWS_PPSM object-group INSIDE_NETS object-group NFS_SERVER log 
```

### Defining the Model.
First lets define our model, The Data Structure below is what will be used to generate the firewall ACL rule set.


### Important Keys

 - **service_groups:** List of objects represents service type object-groups. The *name* property  will be used as the Ports and Protocols for the ACL.
 ```yaml
service_groups:
  - name: NFS_PPSM
    description: PPSM for MGMT network
    ports:
    - number: 111
      protocol: tcp-udp
```

 - **network_groups:** List of objects that represents network type object-groups. The *name* property will be used for either source or destional objects for the acl entry.

 ```yaml
network_groups:
  - name: HQ_MGMT_NETS
    description: Network object for HQ managment networks
    subnets:
    - 10.1.0.0 255.255.255.0
    - 10.1.1.0 255.255.255.0
``` 

 - **acls:** List of acl rule properties. Each ACL can contain multiple *rules*. Within a single rule in the *rules* list, the *status* of [active, inactive]
disables or enables the rule. This is helpful when you would like to pre-stage rules but do not want Ansible to push it to your firewall.
```yaml
acls:
  - name: INSIDE-OUT
    acl_flow: in
    interface: INSIDE
    description: Rules for Internal Networks
    rules:
      - seq: 10
        action: permit
        src_group: INSIDE_NETS
        dest_group: DMZ200_NETS
        acl_ppsm: WEB_PPSM
        status: active
        
    - seq: 20
        action: permit
        src_group: INSIDE_NETS
        dest_group: OUTSIDE_NETS
        acl_ppsm: WEB_PPSM
        status: inactive
```



Below is an example of what all of the above data looks like compile within my hosts/inventory file. In the real world i would have some sort 
of WebGUI that generates these files dynamically based on user input but for this example we will be directly working with the data models and modifying 
the hosts file. 
```yaml
hostname: HQ-FW
ansible_ssh_host: hq-fw.lab.local
ansible_ssh_user: ansible
ansible_ssh_pass: ansible
ansible_become_pass: ansible
ansible_connection: network_cli
ansible_network_os: asa
ansible_become_method: enable
router_id: 10.1.0.2
default_gw: 10.1.2.2

service_groups:
  - name: NFS_PPSM
    description: PPSM for MGMT network
    ports:
    - number: 111
      protocol: tcp-udp
    - number: 1039
      protocol: tcp-udp
    - number: 1047
      protocol: tcp-udp
    - number: 1048
      protocol: tcp-udp
    - number: 2049
      protocol: tcp-udp

network_groups:
  - name: HQ_MGMT_NETS
    description: Network object for HQ managment networks
    subnets:
    - 10.1.0.0 255.255.255.0


acls:
  - name: INSIDE-OUT
    acl_flow: in
    interface: INSIDE
    description: Rules for Internal Networks
    rules:
      - seq: 10
        action: deny
        src_group: INSIDE_NETS
        dest_group: HQ_MGMT_NETS
        acl_ppsm: NFS_PPSM
        status: active

      - seq: 200
        action: permit
        src_group: INSIDE_NETS
        dest_group: HQ_MGMT_NETS
        acl_ppsm: NFS_PPSM
        status: inactive

```
### Road to the Golden config.

Now that we have our model identified we need an template that will use the parameters to create golden configuration. 

```jinja2
{% for acl_rule in acls %}
!
{% for rule in acl_rule.rules %}
access-list {{ acl_rule.name }} remark ###############################################
access-list {{ acl_rule.name }} remark #     Rule for {{ acl_rule.description }}
access-list {{ acl_rule.name }} remark ###############################################
{% if rule.status is defined %}
{% if rule.status == "inactive" %}
no access-list {{ acl_rule.name }} extended {{ rule.action }} object-group {{ rule.acl_ppsm }} object-group {{ rule.src_group  }} object-group {{ rule.dest_group  }} log
{% elif rule.status == "active" %}
access-list {{ acl_rule.name }} line {{ rule.seq }} extended {{ rule.action }} object-group {{ rule.acl_ppsm }} object-group {{ rule.src_group  }} object-group {{ rule.dest_group  }} log
{% endif %}
{% endif %}
{% endfor %}
access-group {{ acl_rule.name }} {{ acl_rule.acl_flow }} interface {{ acl_rule.interface }}
{% endfor %}
```
Generates: 
```text
access-list INSIDE-OUT remark ###############################################
access-list INSIDE-OUT remark #     Rule for Rules for Internal Networks
access-list INSIDE-OUT remark ###############################################
access-list INSIDE-OUT line 10 extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group DMZ200_NETS log
access-list INSIDE-OUT line 20 extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group EXTERNAL_NETS log
access-list INSIDE-OUT line 30 extended permit object-group WINDOWS_PPSM object-group INSIDE_NETS object-group NFS_SERVER log
access-list INSIDE-OUT line 100 extended permit object-group WEB_PPSM object-group INSIDE_NETS object-group ANY_NET log
access-list INSIDE-OUT line 200 extended permit object-group ICMP_PPSM object-group INSIDE_NETS object-group ANY_NET log
access-group INSIDE-OUT in interface INSIDE
```

Now Im sure most of you are looking at this thinking it looks damn intimidating and i will agree to say it was. But learning the Jinja2 templating 
language will be what i believe can set you apart from a standard Ansible network engineers. It is the templating system that allot web developers use within 
most WebDev frameworks and its flexibility allows for an extensive amount of options when it comes to developing your automation workflow. 

The keys and properties all correlate with the data model above. Jinja2 also supports conditionals and loops  as indicated by the snippets located in '{% %}'.
for this demonstration the rule.status property will put a different configuration line item when set accordingly. 

**A litttle nibble:**
special mention for the "*{% if rule.status is defined %}*" line item. **defined** is a keyword used by Jinja2 to validate if the variable is set.
If this line was not present the playbook will fail if the status key is not present. this is a method of making vars optional
in you configuration. 

 


### Preparing the play.

Now first off i learned Ansible maybe 1-2 years ago and haven't touched it until recently, since then i dedicated my time to learning one
interpreted language *python* and one compiled language *golang*. I say this because my approach when going back to Ansible was completely wrong in my opinion lol.
I was trying to write plays as if i was writing scripts or apps and was making it waaaaay more complicated then it needed to be. with that said i want to discuss
a few ansible core modules that will make your life so much easier when writing your plays. 

- **delegate_to:** This command comes in handy when your playbook is set to run on a specific set of hosts but you need a command to run a the localhost or a remotehost.
    I found this to come in handy when im running the template module to store the configs locally. 
    
- **wait_for:** This command is useful when using the *asa_command* or *ios_command* module. This is almost like using expect scripts but 
it offers way more flexibility especially when you start to use Jinja2 conditionals and filters. 

- **with_items:** This command allows you to preform iterations, or loops on list objects in vars. the python equilivant would be 'list comprehensions or standard for loops
    ```python
    acl_list = ["acl1", "acl2"]
    
    def do_something(acl: object):
        return acl
        
    # Standard whileloop   
    for acl in acl_list:
     do_something(acl)
    
    # List comprehension
    result = [do_something(acl) for acl in acl_list]
    ```
- **loop_control.label:** This command is especially helpful for controlling CLI noise when using the *ansible-playbook* command when a task is preforming loop iteration. 
This will allow you to access a key of an item your iterating through and it will display as the item name when the task is executing. Without a label being set ansible will return the entire 
data structure. which can be allot of jumbled text if you have complex data structures you are working with. 
    ```yaml
    tasks:
    - name: Checking {{ ansible_ssh_host }} if ACLs are actively on the device.
      become: yes
      asa_command:
        commands:
          - show run object-group id {{ item.name }}
      with_items:
        - "{{ network_objects }}"
      loop_control:
        label: "{{ item.name }}"  
  
    ```
    Results: 
    ```text
    TASK [Checking if Network Object-Groups are present.] **************************
    ok: [hq-fw] => (item=HQ_MGMT_NETS)
    ok: [hq-fw] => (item=INSIDE_NETS)
    ok: [hq-fw] => (item=DMZ200_NETS)
    ok: [hq-fw] => (item=ANY_NET)
    ok: [hq-fw] => (item=NFS_SERVER)
    ok: [hq-fw] => (item=EXTERNAL_NETS)
    ```

- **register:** This is used when you would like to store the variable of the executed task to more verifiable variable name. This command can help 
make your playbooks more readable and easier to follow was the default stdout can be confusing no beginning ansible users. 