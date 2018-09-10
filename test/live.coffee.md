The plan
========

    chai = require 'chai'
    chai.should()

    Promise = require 'bluebird'
    path = require 'path'
    current_dir = path.dirname __filename
    process.chdir current_dir
    pkg = require '../package.json'
    debug = (require 'debug') "#{pkg.name}:test:live"
    seconds = 1000
    path = require 'path'
    PouchDB = require 'ccnq4-pouchdb'

    fs = Promise.promisifyAll require 'fs'
    request = require 'superagent'

    exec = (require 'exec-as-promised')()

    describe 'Live test', ->

      default_interval = 10*seconds # keep it short so that we don't have to wait forever for the test suite to finish!!
      default_host = 'phone.example.net'

      config = (name) ->
        provisioning: "http://couchdb:5984/db_#{name}"
        provisioning_admin: "http://couchdb:5984/db_#{name}"
        interval: default_interval
        host: default_host

      record =
        _id:'number:33987654321'
        type:'number'
        number:'33987654321'
        account:'test'
        registrant_username:'foo'
        registrant_password:'bar'
        registrant_host: default_host

      before ->
        @timeout 90*seconds
        debug "#{pkg.name} live tester: Waiting #{delay} seconds for image to be ready."
        await Promise.delay delay*seconds
        {body} = await request
          .get 'http://couchdb:5984'
          .accept 'json'
        debug "CouchDB #{body.version} is ready."

      restart = (done) ->
        start = new Date()
        (_,cancel) ->
          stop = new Date()
          delay = stop-start
          debug 'restart delay', delay
          if delay > default_interval
            done()
          else
            done new Error "Delay too short, observed = #{delay}, expected = #{default_interval}"
          await Promise.delay(500)
          cancel?()
          null

      run = require '..'

      name = 1

      it 'should not report on no changes', (done) ->
        @timeout 15*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record

          # uut
          await run (-> done new Error 'Should not have called'), cfg

          # check
          await Promise.delay default_interval+2*seconds
          done()
        null

      it 'should not report on put with no changes', (done) ->
        @timeout 15*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record

          # uut
          await run (-> done new Error 'Should not have called'), cfg

          # trigger
          doc = await db.get record._id
          await db.put doc
          await Promise.delay default_interval+2*seconds
          done()
        null

      it 'should report on creation', (done) ->
        @timeout 11*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning

          # uut
          await run (restart done), cfg

          # trigger
          await db.put record
        null

      it 'should report on update', (done) ->
        @timeout 11*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record

          # uut
          await run (restart done), cfg

          # trigger
          doc = await db.get record._id
          doc.registrant_password = 'boo'
          await db.put doc
        null

      it 'should report on put with no changes preceded by changes', (done) ->
        @timeout 12*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record

          # uut
          await run (restart done), cfg

          # trigger
          doc = await db.get record._id
          doc.registrant_password = 'bar'
          await db.put doc
          doc = await db.get record._id
          doc.registrant_password = 'boo'
          await db.put doc
        null

      it 'should report on deletion', (done) ->
        @timeout 12*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record

          # uut
          await run (restart done), cfg

          # trigger
          doc = await db.get record._id
          doc._deleted = true
          await db.put doc
        null

      it 'should report on creation after deletion', (done) ->
        @timeout 12*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record
          doc = await db.get record._id
          doc._deleted = true
          await db.put doc

          # uut
          await run (restart done), cfg

          # trigger
          await Promise.delay 500
          await db.put
            _id:'number:33987654321'
            type:'number'
            number:'33987654321'
            account:'test2'
            registrant_username:'test2'
            registrant_password:'foobar'
            registrant_host: default_host
        null

      it 'should report on update after deletion', (done) ->
        @timeout 12*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record
          doc = await db.get record._id
          doc._deleted = true
          await db.put doc

          # uut
          await run (restart done), cfg

          # trigger
          await db.put record
        null

      it 'should report on deletion after deletion', (done) ->
        @timeout 12*seconds
        do ->
          # prep
          cfg = config name++
          db = new PouchDB cfg.provisioning
          await db.put record
          doc = await db.get record._id
          doc._deleted = true
          await db.put doc

          # uut
          await run (restart done), cfg

          # trigger
          doc =
            _id: record._id
            _deleted: true
          await db.put doc
        null

    describe 'When using a config file', ->
      docker =
        image: "shimaore/couchdb:1.6.1"
        container: "#{pkg.name}-test-container-2"
        name: "#{pkg.name}-test-instance-2"

      before ->
        @timeout 90*seconds
        delay = 2
        await exec "docker rm #{docker.container}"
          .catch -> true
        await exec "docker run -d --net=host --name #{docker.container} #{docker.image}"

        debug "#{pkg.name} live tester: Waiting #{delay} seconds for image to be ready."
        await Promise.delay delay*seconds
        {body} = await request
          .get 'http://couchdb:5984'
          .accept 'json'
        debug "CouchDB #{body.version} is ready."

      after ->
        await Promise.delay 500
        await exec "docker kill #{docker.container}"

      it 'should write a proper config file', ->
        run = require '..'
        config =
          provisioning: "http://couchdb:5984/db_bear"
          provisioning_admin: "http://couchdb:5984/db_bear"
          host: 'example.net'

        process.env.CONFIG = './config.json'
        await fs.writeFileAsync process.env.CONFIG, (JSON.stringify config), encoding:'utf8'
        debug 'wrote'

        {cancel} = await run ->

        buf = await fs.readFileAsync process.env.CONFIG, encoding:'utf8'
        debug 'got', buf
        data = JSON.parse buf
        data.should.have.property 'provisioning', config.provisioning
        data.should.have.property 'provisioning_admin', config.provisioning_admin
        data.should.have.property 'host', config.host
        data.should.have.property 'update_seq'

        cancel?()

        await fs.unlinkAsync process.env.CONFIG
