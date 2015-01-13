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
      assert local_host?, "#{pkg.name}: needed: missing localhost"
      assert old_doc?, "#{pkg.name}: needed: missing old_doc"
      assert new_doc?, "#{pkg.name}: needed: missing new_doc"
      changed = false

      value = (doc,name) ->
        return null unless doc.account?
        return null if doc._deleted or doc.disabled
        host = doc.registrant_host?.split(':')[0]
        return null unless host is local_host
        doc[name] ? null

      for name in fields
        do (name) ->
          old_value = value old_doc, name
          new_value = value new_doc, name
          if old_value isnt new_value
            changed = true
      changed

    module.exports = needed
    assert = require 'assert'
    pkg = require './package.json'
