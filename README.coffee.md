A process that restarts the local registrant when needed
========================================================

First let's define what we mean by "when needed". We mean that changes were brought to any of the `registrant_*` fields in the global-number document that references them.

    fields = [
      'registrant_expiry'
      'registrant_host'
      'registrant_password'
      'registrant_realm'
      'registrant_remote_ipv4'
      'registrant_socket'
      'registrant_username'
    ]

However since calls can't be placed if the `account` field is empty, we don't need to be alerted of changes in that case. Also, obviously the changes only apply to a host that is listed as the `registrant_host`.

    needed = (local_host,old_doc,new_doc) ->
      changed = false

      value = (doc,name) ->
        return null unless doc.account?
        host = doc.registrant_host.split(':')[0]
        return null unless host is local_host
        doc[name] ? null

      for name in fields
        do (name) ->
          old_value = value old_doc, name
          new_value = value new_doc, name
          if old_value isnt new_value
            changed = true
      changed

Test!

    assert = require 'assert'
    assert (not needed 'a', {account:1, registrant_host:'b'}, {account:1,registrant_host:'c'}), 'Host mismatch'
    assert (not needed 'a', {account:1, registrant_host:'a'}, {account:1,registrant_host:'a'}), 'Host match but no data'
    assert (needed 'a', {account:1, registrant_host:'a'}, {account:1,registrant_host:'b'}), 'Host changed'
    assert (not needed 'a', {account:1, registrant_host:'a:5070'}, {account:1,registrant_host:'a:5070'}), 'Host with port'
    assert (needed 'a', {account:1, registrant_host:'a:5070'}, {account:1,registrant_host:'a:5080'}), 'Host with port changed'
    assert (not needed 'a', {account:1, registrant_host:'a',registrant_password:'foo'}, {account:2, registrant_host:'a',registrant_password:'foo'}), 'Host match but no changes'
    assert (needed 'a', {account:1, registrant_host:'a',registrant_password:'foo'}, {account:2, registrant_host:'a',registrant_password:'bar'}), 'Simple change'
    assert (not needed 'a', {account:1,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2, registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}), 'No changes in complex set'
    assert (needed 'a', {account:1,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:lo'}), 'Changes in complex set'
    assert (needed 'a', {registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2, registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}), 'Account was added'
