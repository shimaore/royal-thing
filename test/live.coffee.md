The plan
========

    chai = require 'chai'
    chai.should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    pkg = require '../package.json'
    debug = (require 'debug') "#{pkg.name}:test:live"
    seconds = 1000
    CouchDB = require 'most-couchdb'
    most = require 'most'
    {EventEmitter} = require 'events'

    describe 'Live test', ->

      default_interval = 7*seconds # keep it short so that we don't have to wait forever for the test suite to finish!!
      default_host = 'phone.example.net'

      stop = ->
        ev = new EventEmitter()
        limit: most.fromEvent('end', ev)
        close: -> ev.emit 'end'

      config = (name) ->
        provisioning: "http://admin:password@couchdb:5984/db_#{name}"
        provisioning_admin: "http://admin:password@couchdb:5984/db_#{name}"
        interval: default_interval
        host: default_host

      create = (db) ->
        await db.destroy().catch -> yes
        await db.create()

      record =
        _id:'number:33987654321'
        type:'number'
        number:'33987654321'
        account:'test'
        registrant_username:'foo'
        registrant_password:'bar'
        registrant_host: default_host

      restart = ->
        start_time = new Date()
        error = new Error "Was never called."
        handler: ->
          stop_time = new Date()
          delay = stop_time-start_time
          debug 'restart delay', delay
          if delay > default_interval
            error = null
          else
            error = new Error "Delay too short, observed = #{delay}, expected = #{default_interval}"
          null
        errored: ->
          throw error if error

      run = require '..'

      name = 1

      it 'should not report on no changes', ->
        @timeout 15*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record

        # uut
        error = null
        {limit,close} = stop()
        {completed} = await run ( (error) -> error = new Error "Should not have called: #{error}"), cfg, limit

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        throw error if error

      it 'should not report on put with no changes', ->
        @timeout 15*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record

        # uut
        error = null
        {limit,close} = stop()
        {completed} = await run ( (error) -> error = new Error "Should not have called: #{error}"), cfg, limit

        # trigger
        doc = await db.get record._id
        await db.put doc

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        throw error if error

      it 'should report on creation', ->
        @timeout 11*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        await db.put record

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it 'should report on update', ->
        @timeout 11*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        doc = await db.get record._id
        doc.registrant_password = 'boo'
        await db.put doc

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it 'should report on put with no changes preceded by changes', ->
        @timeout 12*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        doc = await db.get record._id
        doc.registrant_password = 'bar'
        await db.put doc
        doc = await db.get record._id
        doc.registrant_password = 'boo'
        await db.put doc

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it 'should report on deletion', ->
        @timeout 12*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        doc = await db.get record._id
        doc._deleted = true
        await db.put doc

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it 'should report on creation after deletion', ->
        @timeout 12*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record
        doc = await db.get record._id
        doc._deleted = true
        await db.put doc

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        await sleep 500
        await db.put
          _id:'number:33987654321'
          type:'number'
          number:'33987654321'
          account:'test2'
          registrant_username:'test2'
          registrant_password:'foobar'
          registrant_host: default_host

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it 'should report on update after deletion', ->
        @timeout 12*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record
        doc = await db.get record._id
        doc._deleted = true
        await db.put doc

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        await db.put record

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()

      it.skip 'should report on deletion after deletion', ->
        @timeout 12*seconds
        # prep
        cfg = config name++
        db = new CouchDB cfg.provisioning
        await create db
        await db.put record
        doc = await db.get record._id
        doc._deleted = true
        await db.put doc

        # uut
        {handler,errored} = restart()
        {limit,close} = stop()
        {completed} = await run handler, cfg, limit

        # trigger
        doc =
          _id: record._id
          _deleted: true
        await db.put doc

        # check
        await sleep default_interval+2*seconds
        close()
        await completed
        errored()
