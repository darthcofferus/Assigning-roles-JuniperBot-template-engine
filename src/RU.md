```Nunjucks
{# Автор: Severus3107 aka Darth Cofferus #}

{# Версия: 1.0 #}

{# Этот код будет корректно работать как в шаблоне сообщения, так и в шаблоне панели. Для красивого внешнего вида рекомендуется использовать второй вариант. #}

{# Чтобы Джунипер мог выдавать роли, у него должны быть соответствующие права и его роль должна находиться ВЫШЕ выдаваемых ролей. #}

{# Для этой команды рекомендуется выбрать канал в настройках (КОМАНДА -> Разрешенные каналы и категории). #}

{# ! Параметры, требующие обязательного редактирования ! #}

{% set uuid = 'uuid' %} {# Скопируйте сюда UUID действия, в которое вставили этот код.
        Вы можете получить этот UUID в настройках действия (нажмите иконку "i"). #}

{% global ids = ['id', 'id'] %} {# Список идентификаторов ролей для выдачи, вставьте сюда свой
        список. Список может содержать любое количество элементов. #}

{# Дополнительные параметры #}

{% global singleMod = true %} {# Режим выдачи ролей.
        true: у пользователя одновременно может быть только одна роль из списка, если он выберет другую, то старая роль удалится.
        false: пользователь может одновременно иметь неограниченное количество ролей из списка. #}

{% set f = '**' %} {# Джунипер добавит эти символы для форматирования текста в начало и конец каждого сообщения,
        измените по желанию. Может быть пустым (две одинарных/двойных кавычки БЕЗ пробела между ними) или null. #}

{% set description = null %} {# Текст в корневом сообщении (измените по желанию, может быть пустым (две одинарных/двойных кавычки) или null). #}

{% set imageLink = null %} {# Ссылка на картинку в корневом сообщении (измените по желанию, может быть null). #}

{# ! ВНИМАНИЕ! НЕ редактируйте написанный ниже код, если вы ничего не понимаете в программировании ! #}

{% require (uuid is not null) and (uuid != 'uuid') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ Вы не указали UUID действия!'~f).error() %}

{% require (ids is not null) and (ids is not empty) and (ids != '[id, id]') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ Вы не указали идентификаторы выдаваемых ролей!'~f).error() %}

{% require bot.hasPermission('MANAGE_ROLES') returning sourceChannel.createEmbed()
        .withDescription(f~'❌ У бота нет прав на выдачу ролей!'~f).error() %}

{% global cId = component.id %}

{% macro getListForDeletion() %} {# 2 использования #}
  {% set listForDeletion = [] %}
  {% for id in ids %}
    {% if id != cId and member.hasRole(id) %}
      {% do listForDeletion.add(id) %}
    {% endif %}
  {% endfor %}
  {% return listForDeletion %}
{% endmacro %}

{% macro getNumberAndNamesOfDeletedRoles(listForDeletion) %} {# 2 использования #}
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
  {% return format('%s %s %s', count, plural(length, 'ru', 'роль', 'роли', 'ролей', 'роли'), result) %}
{% endmacro %}

{% macro replaceCId(str) %} {# 2 использования #}
  {% if startsWith(cId, str) %}
    {% do (map := {}).put(str, '') %}
    {% set cId = replace(cId, map) %}
    {% return true %}
  {% else %}
    {% return false %}
  {% endif %}
{% endmacro %}

{% if cId == 'remove' %} {# Этот код выполняется при нажатии на кнопку "Удалить роли" в корневом сообщении. #}
  {% do override.preferEphemeral(true).withColor('ec4a48') %}
  {% set length = length(listForDeletion := getListForDeletion()) %}
  {% if length > 0 %}
    {% do member.modifyRoles([role.id], listForDeletion) %}
    {{f}}❎ Вы удалили{{ getNumberAndNamesOfDeletedRoles(listForDeletion) }}!{{f}}
  {% else %}
  {{f}}❌ У вас нет ролей, которые можно было бы удалить.{{f}}
  {% endif %}

{% elseif (notNull := cId is not null) and length(cId) < 19 and (cId == 0 or
        (nextIsPressed := replaceCId('next')) or
        (lastIsPressed := replaceCId('last'))) %} {# Этот код выполняется при нажатии на кнопку "Получить роли" в
        корневом сообщении или при нажатии на кнопки переключения страниц в меню выбора ролей. #}

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
  {{f}}📑 Страница {{ current }}/{{ max }}{{f}}

{% elseif notNull %} {# Этот код выполняется при нажатии на кнопку с ролью в меню выбора ролей. #}

  {% set role = guild.getRole(component.id) %}
  {% do override.withColor(role.color).preferEphemeral(true) %}
  {% require role returning sourceChannel.createEmbed()
          .withDescription(f~'❌ Данной роли больше не существует!'~f).error() %}

  {% if member.hasRole(role) %}
    {% do member.removeRole(role) %}
    {{f}}❎ Вы удалили роль {{ role.name }}!{{f}}
  {% else %}
    {% set listForDeletion = singleMod ? getListForDeletion() : [] %}
    {% do member.modifyRoles([role.id], listForDeletion) %}
    {% set end = (length(listForDeletion) > 0) ? ' и удалили ' ~ getNumberAndNamesOfDeletedRoles(listForDeletion) : '' %}
    {{f}}✅ Вы получили роль {{ role.name }}{{ end }}!{{f}}
 {% endif %}

{% else %} {# Этот код выполняется при отправке команды. #}

  {% do message.delete() %}
  {% set embed = sourceChannel.createEmbed().withTitle(f~'-◈-ㅤВыдача ролейㅤ-◈-'~f).withColor('B100FF').withDescription(description)
        .withFooter('https://github.com/darthcofferus', 'https://media.discordapp.net/attachments/1314676835199746069/1315221058839646208/developer.png?' ~
                    'ex=67569e90&is=67554d10&hm=fd7b3137480e6d25460a73927623e29c774ee6304a3c77d5be05d2cce8fcbebf&=&format=webp&quality=lossless&' ~
                    'width=670&height=670').withImage(imageLink) %}
  {% do embed.addButton('SUCCESS', 0, 'Получить роли', null, uuid) %}
  {% do embed.addButton('DANGER', 'remove', 'Удалить роли', null, uuid) %}
  {% do embed.send() %}

{% endif %}
```
