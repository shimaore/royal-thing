Test!

    assert = require 'assert'

    describe 'Needed', ->
      needed = require '../needed'
      it 'should not break if no host is present', ->
        assert not needed 'a', {}, {}
      it 'should not report on deleted documents that remain deleted', ->
        assert not needed 'a', {account:1, registrant_host:'a', registrant_username:'foo', _deleted:true}, {account:1,registrant_host:'a', registrant_username:'bar',_deleted:true}
      it 'should report on created documents', ->
        assert needed 'a', {account:1, registrant_host:'a', registrant_username:'foo', _deleted:true}, {account:1,registrant_host:'a', registrant_username:'foo'}
      it 'should report on deleted documents', ->
        assert needed 'a', {account:1, registrant_host:'a', registrant_username:'foo'}, {account:1,registrant_host:'a', registrant_username:'foo',_deleted:true}
      it 'should not report a host mismatch', ->
        assert not needed 'a', {account:1, registrant_host:'b'}, {account:1,registrant_host:'c'}
      it 'should not report a host match without data', ->
        assert not needed 'a', {account:1, registrant_host:'a'}, {account:1,registrant_host:'a'}
      it 'should report that the host changed', ->
        assert needed 'a', {account:1, registrant_host:'a'}, {account:1,registrant_host:'b'}
      it 'should not report that the host has a port', ->
        assert not needed 'a', {account:1, registrant_host:'a:5070'}, {account:1,registrant_host:'a:5070'}
      it 'should report that the port changed', ->
        assert needed 'a', {account:1, registrant_host:'a:5070'}, {account:1,registrant_host:'a:5080'}
      it 'should not report that the host matches but no changes were made', ->
        assert not needed 'a', {account:1, registrant_host:'a',registrant_password:'foo'}, {account:2, registrant_host:'a',registrant_password:'foo'}
      it 'should report a simple change', ->
        assert needed 'a', {account:1, registrant_host:'a',registrant_password:'foo'}, {account:2, registrant_host:'a',registrant_password:'bar'}
      it 'should not report lack of changes in a complex set', ->
        assert not needed 'a', {account:1,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2, registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}
      it 'should report changes in complex set', ->
        assert needed 'a', {account:1,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2,registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:lo'}
      it 'should report an account was added', ->
        assert needed 'a', {registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}, {account:2, registrant_host:'a',registrant_password:'foo',registrant_socket:'sip:ka'}
