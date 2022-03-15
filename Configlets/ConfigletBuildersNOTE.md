# ACE5Lab

## Notes on CVP CB

A classical CB script, would start with these lines

<pre>from cvplibrary import CVPGlobalVariables, GlobalVariableNames
import yaml 
from cvplibrary import RestClient
import ssl

global_vars = CVPGlobalVariables.getValue(GlobalVariableNames.CVP_SYSTEM_LABELS)</pre>

We also need to open a exception. (still needs work here for proper hardening)
<pre>ssl._create_default_https_context = ssl._create_unverified_context</pre>

If we print global_vars for a given device, let's say leaf1-DC1, this
returns a list of strings:

<pre>['hostname:leaf1-DC1',
 'serialnumber:leaf1-DC1',
 'topology_type:leaf',
 'tapagg:none',
 'eostrain:4.26',
 'bgp:enabled',
 'model:cEOSLab',
 'terminattr:v1.14.0',
 'ztp:false',
 'eos:4.26.0F',
 'topology_rack:pod1',
 'topology_pod:OOB-DC1-spine1-DC1-spine2-DC1-spine3-DC1',
 'mpls:false',
 'systype:fixed',
 'Container:pod1',
 'Container:Tenant',
 'Container:ATD_LEAFS',
 'Container:ATD_FABRIC']</pre>

<p>Then, it's possible to iterate and match over GlobalVars. We can store the hostname with:<br>
<pre>for var_item in global_vars:
    item, value = var_item.split(':')
    if item == 'hostname':
        hostname = value</pre></p>
 
<p>Let's consider a following list, created as a simple configlet
(and not directly attached to any containers)
<pre>VLAN_List
leaf1-DC1:
    vlans:
        100:
            name: Customer100
        200:
            name: Customer200
leaf2-DC1:
    vlans:
        100:
            name: Customer100
        200:
            name: Customer200</pre></p>

As we can see, this has a list of VLANs in a human readable format.

Then, let's assume we want to have this Vlans configured only in each device, i.e. Leaf1-DC1 and Leaf2-DC1

First we need to connect CVP, even internally. So, we call a method like:

vlan_client = RestClient('https\://192.168.0.5/cvpservice/configlet/getConfigletByName.do?name=VLAN_List','GET')

Where the url provided is pulling the configlet into the method.

We can make some variables, so changing the code will be easier 
<p><pre>cvp_ip = '192.168.0.5'
configlet_name = 'VLAN_list'
vlan_client = RestClient( 'https\://' + cvp_ip + 
                          '/cvpservice/configlet/getConfigletByName.do?name=' +
                          configlet_name , 'GET' )
if vlan_client.connect():
    raw = yaml.load(client.getResponse())</pre>

The raw will have the following dict structure
<pre>{'reconciled': False,
 'netElementCount': 0,
 'editable': True,
 'visible': True,
 'containerCount': 0,
 'user': 'arista',
 'key': 'configlet_084d89f9-fc9d-4506-bf6b-5d30f24aaf48',
 'sslConfig': False,
 'typeStudioConfiglet': False,
 'name': 'VLAN_List',
 'config': 'leaf1-DC1:\n    vlans:\n        100:\n            name: Customer100\n        200:\n            name: Customer200\nleaf2-DC1:\n    vlans:\n        100:\n            name: Customer100\n        200:\n            name: Customer200',
 'dateTimeInLongFormat': 1647292335169,
 'isDraft': False,
 'note': '',
 'type': 'Static',
 'isDefault': 'no',
 'isAutoBuilder': ''}</pre>

We want to extract only the config contents, a YAML text, but also converted in a
dict format, so we also do a yaml load on this value.

<pre>vlans_data_model = yaml.safe_load(raw['config'])</pre>

And then we will have a nice dict data model
<pre>{'leaf1-DC1': 
 {'vlans': 
  {200: 
    {'name': 'Customer200'}, 
   100: 
    {'name': 'Customer100'}}},
 'leaf2-DC1':
 {'vlans':
  {200:
    {'name': 'Customer200'},
   100:
    {'name': 'Customer100'}}}}</pre>

And this is how we can iterate: we can push only if the hostname is listed on VLAN_List
<pre>for device in vlans_data_model:
    if device == hostname:
        for vlan_id in vlans_data_model[device]['vlans']:
            print('vlan ' + str(vlan_id))
            print('  name ' + vlans_data_model[device]['vlans'][vlan_id]['name'] )</pre>
