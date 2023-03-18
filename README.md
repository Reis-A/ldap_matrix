# Matrix Corporal Policy Specification Using LDAP Groups
 
* This scripts takes as input a specification in the form of a YAML file  and generates a so-called *Policy* file for Matrix Corporal reconciliator tool (see https://github.com/devture/matrix-corporal/blob/master/docs/policy.md) with data coming from an LDAP server in which groups are specified containing `memberUid` entries.
* It also generates any rooms and spaces mentioned in the yaml file, that do not exist on the Synapse Homeserver yet.
* It also assigns powerlevels to ldapusers in managed rooms

## Install with Pipenv

* install `pipenv` tool as a user (https://pipenv.pypa.io)
* `$ export PATH=$HOME/.local/bin/:$PATH`
* `$ pipenv sync`
* `$ pipenv shell`
* `$pipenv run ./spec2policy input.yml policy.json`

## Install with Pip (not recommended)

* Python >= 3.6
* `pip install -r requirements.txt`


## Configuration
   * configure in `$HOME/.ldapsync.cfg` your server credentials (see `ldapsync.cfg` as example)
   * change (at least) DOMAIN and LDAP_SEARCH_BASE (potentially LDAP filters also) in `spec2policy.py`
   * (optional) If the shared secret provider is installed on the HomeServer, you can use it to create the ADMINAUTH Token. 
                 The ADMINAUTH token can also be set manually by editing it in the ldapcfg file.
   *  For the room creation process to work it is recommended to access the homeserver directly thru port 8008 and not via matrix-corporal
## Usage
*  To generate the policy file, type: `$ ./spec2policy.py input.yml policy.json`
*  to push the policy to matrix-corporal, type `$ curl -s --insecure -XPUT --data "@$(pwd)/policy.json" -H 'Authorization: Bearer ......' https://matrix.domain.com/_matrix/corporal/policy | jq .`
* also to automate the build and deployment of the policy file, it is possible to use a CI tool such as Jenkins, Gitlab CI, Travis CI etc. See the `.gitlab-ci.yml` as an example (it requires the definition of the credentials in a protected $CFG environment variable).

## Input specification format

~~~
---
ldap-matrix: version 0.3 
#here set flags on user level based on LDAP Group memberships:
ldapgroups-forbidroomcreation:
   - ldapgroup1
   - ldapgroup2
ldapgroups-forbidencryptedroomcreation:
   - ldapgroup1
ldapgroups-forbidunencryptedroomcreation:
   - ldapgroup2
---
#list of spaces and rooms. 
#Rooms can be attached to a space with setting the entry childof
# In this example one private space "Spacename" with the private room "Roomname" will be created plus the corresonding policy.json for matrix-corporal 
- space: SpaceName
  ldapgroups:
    - ldapgroup1: 50 #this means ldapgroup1 has in space SpaceName powerlevel 50 (default Moderator)
    - ldapgroup2     # no powerlevel is set -> default behavior
  ldapusers:
    - supplementaryuser1: 0  #powerlevel is set to 0 and previous powerlevel is overwritten by 0 if allowed
    - supplementaryuser2
 - room: Roomname
   ldapgroups:
    - ldapgroup1
   ldapusers:
    - ldapuser1: 100 # this means supplementary user2 gets powerlevel 100 and is roomadmin of the room Roomname
   childof: SpaceName 
~~~
### Ldapgroup based flags
* On the first part of the yaml file, set the user based flags for the members of *ldapgroups-forbidroomcreation* and *ldapgroups-forbidencryptedroomcreation* and *forbidunencryptedroomcreation*. 
If no userbased flags are set, the flags are globally set to false in the policy file. 

### Creation of Rooms and Spaces for LDAPgroups
* On the second part of the yaml file, list all spaces/rooms with their ldapgroups, individual ldapusers and with *childof* the parent space of each room, that you want to place in a space. 
* All the rooms and spaces are created as private rooms by default. This can be adjusted by modifying the appropriate functions in spec2policy.py 

### Assign Powerlevels by LDAPgroup memberships

For any ldapuser entry or ldapgroup entry of a room or space, one can optionally set the powerlevel.
Ldap_matrix reads the current powerlevels of each managed room and updates the powerlevel entrys with the settings in the yaml file.
If no powerlevel is set in the yaml file, ldap_matrix will make no changes to the powerlevels in the room. Be aware that once a user is given powerlevel 100, the powerlevel cannot be reduced easily anymore, since the roomadmin has the same powerlevel. 


## Caveats

* There is no room deletion immplemented nor planned. To delete rooms, use the synapse admin gui, for instance.
* A Matrix Administrator Account has to be enrolled in all of the managed rooms.
 This is necessary for the conciliation of matrix-corporal to work. Since all the managed rooms are created by the admin account, this condition is automatically satisfied.

## Future ideas:
* implement hooks maybe with own API to better control room memberships.
my goal: rooms are only allowed if at least one member of a certain ldap group ("teachers") is a member of the room.
i think it is possible to do, but needs the implementation of an external REST API for matrix-corporal to check

