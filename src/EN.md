```Nunjucks
{# Author: Severus3107 aka Darth Cofferus #}

{# Version: 1.0 #}

{# This code will work correctly in both the message template and the panel template. For a beautiful appearance, it is recommended to use the second option. #}

{# In order for Juniper to assign roles, he must have the appropriate rights and his role must be ABOVE the roles assigned. #}

{# For this command, it is recommended to select a channel in the settings (COMMAND -> Allowed Channels and Categories). #}

{# ! Parameters that require mandatory editing ! #}

{% set uuid = 'uuid' %} {# Copy here the UUID of the action where you inserted this code.
        You can get this UUID in the action settings (click the "i" icon). #}

{% global ids = ['id', 'id'] %} {# List of role IDs to be issued, insert your own here
        list. The list can contain any number of elements. #}

{# Additional parameters #}

{% global singleMod = true %} {# Role assignment mode.
        true: A user can have only one role from the list at a time, if they select another, the old role will be deleted.
        false: A user can simultaneously have an unlimited number of roles from the list. #}

{% set f = '**' %} {# Juniper will add these text formatting characters to the beginning and end of each message.
        Сhange it as desired. It can be empty (two single/double quotes WITHOUT a space between them) or null. #}

{% set description = null %} {# The text in the root message (change as desired, can be empty (two single/double quotes) or null). #}

{% set imageLink = null %} {# The link to the image in the root message (change it if desired, maybe null). #}

{# ! ATTENTION! DO NOT edit the code below if you do not understand anything about programming ! #}

{% require (uuid is not null) and (uuid != 'uuid') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ You did not specify the UUID of the action!'~f).error() %}

{% require (ids is not null) and (ids is not empty) and (ids != '[id, id]') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ You did not specify the IDs of the assigned roles!'~f).error() %}

{% require bot.hasPermission('MANAGE_ROLES') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ The bot does not have the rights to assign roles!'~f).error() %}

{% global cId = component.id %}

{% macro getListForDeletion() %} {# 2 usages #}
  {% set listForDeletion = [] %}
  {% for id in ids %}
    {% if id != cId and member.hasRole(id) %}
      {% do listForDeletion.add(id) %}
    {% endif %}
  {% endfor %}
  {% return listForDeletion %}
{% endmacro %}

{% macro getNumberAndNamesOfDeletedRoles(listForDeletion) %} {# 2 usages #}
  {% set result = '' %}
  {% for id in listForDeletion %}
    {% set role = guild.getRole(id) %}
    {% if (length := length(listForDeletion)) > 1 and not (loop.last) %}
      {% set result = result ~ role.name ~ ', ' %}
    {% else %}
      {% set result = result ~ role.name %}
    {% endif %}
  {% endfor %}
  {% if length > 1 %}
    {% set result = '(' ~ result ~ ')' %}
  {% endif %}
  {% set count = (length > 1) ? length : '' %}
  {% return format('%s %s %s', count, plural(length, 'en', 'role', 'roles'), result) %}
{% endmacro %}

{% macro replaceCId(str) %} {# 2 usages #}
  {% if startsWith(cId, str) %}
    {% do (map := {}).put(str, '') %}
    {% set cId = replace(cId, map) %}
    {% return true %}
  {% else %}
    {% return false %}
  {% endif %}
{% endmacro %}

{% if cId == 'remove' %} {# This code is executed when you click on the "Delete roles" button in the root message. #}
  {% do override.preferEphemeral(true).withColor('ec4a48') %}
  {% set length = length(listForDeletion := getListForDeletion()) %}
  {% if length > 0 %}
    {% do member.modifyRoles([role.id], listForDeletion) %}
    {{f}}❎ You deleted the{{ getNumberAndNamesOfDeletedRoles(listForDeletion) }}!{{f}}
  {% else %}
  {{f}}❌ You don't have any roles that can be deleted.{{f}}
  {% endif %}

{% elseif (notNull := cId is not null) and length(cId) < 19 and (cId == 0 or
        (nextIsPressed := replaceCId('next')) or
        (lastIsPressed := replaceCId('last'))) %} {# This code is executed by clicking on the "Get roles" button in
        the root message or by clicking on the page switching buttons in the role selection menu. #}

  {% do override.preferEphemeral(true).withColor('3ba55b') %}
  {% set i = 1 %}
  {% set indexOfLastButton = 0 %}
  {% for id in ids %}
    {% set index = ids.indexOf(id) %}
    {% if (nextIsPressed and index < cId) or (lastIsPressed and index < cId - 23 and cId > 25) %}
      {% continue %}
    {% endif %}
    {% set role = guild.getRole(id) %}
    {% if role is not null %}
      {% set indexOfLastButton = index %}
      {% if i == 25 and length(ids) > 25 %}
        {% do override.addButton('PRIMARY', 'next' ~ index, null, '➡️', uuid) %}
        {% break %}
      {% endif %}
      {% if i == 1 and index > 0 %}
        {% do override.addButton('PRIMARY', 'last' ~ index, null, '⬅️', uuid) %}
        {% set i = i + 1 %}
      {% endif %}
      {% do override.addButton('SECONDARY', id, role.name, null, uuid) %}
      {% set i = i + 1 %}
    {% endif %}
  {% endfor %}
  {% do override.editComponentMessage(nextIsPressed or lastIsPressed) %}
  {% set max = (length(ids) <= 25) ? 1 : round(length(ids) / 25, 'CEIL') %}
  {% set current = (length(ids) <= 25) ? 1 : round(indexOfLastButton / 25, 'CEIL') %}
  {{f}}📑 Page {{ current }}/{{ max }}{{f}}

{% elseif notNull %} {# This code is executed by clicking on the role button in the role selection menu. #}

  {% set role = guild.getRole(component.id) %}
  {% do override.withColor(role.color).preferEphemeral(true) %}
  {% require role returning sourceChannel.createEmbed()
          .withDescription(f~'❌ This role no longer exists!'~f).error() %}

  {% if member.hasRole(role) %}
    {% do member.removeRole(role) %}
    {{f}}❎ You deleted the role {{ role.name }}!{{f}}
  {% else %}
    {% set listForDeletion = singleMod ? getListForDeletion() : [] %}
    {% do member.modifyRoles([role.id], listForDeletion) %}
    {% set end = (length(listForDeletion) > 0) ? ' and deleted ' ~ getNumberAndNamesOfDeletedRoles(listForDeletion) : '' %}
    {{f}}✅ You got the role {{ role.name }}{{ end }}!{{f}}
 {% endif %}

{% else %} {# This code is executed when sending a command. #}

  {% do message.delete() %}
  {% set embed = sourceChannel.createEmbed().withTitle(f~'-◈-ㅤAssigning rolesㅤ-◈-'~f).withColor('B100FF').withDescription(description)
        .withFooter('https://github.com/darthcofferus', 'https://media.discordapp.net/attachments/1314676835199746069/1315221058839646208/developer.png?' ~
                    'ex=67569e90&is=67554d10&hm=fd7b3137480e6d25460a73927623e29c774ee6304a3c77d5be05d2cce8fcbebf&=&format=webp&quality=lossless&' ~
                    'width=670&height=670').withImage(imageLink) %}
  {% do embed.addButton('SUCCESS', 0, 'Get roles', null, uuid) %}
  {% do embed.addButton('DANGER', 'remove', 'Delete roles', null, uuid) %}
  {% do embed.send() %}

{% endif %}
```
