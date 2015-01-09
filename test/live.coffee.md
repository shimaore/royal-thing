The plan
========

    chai = require 'chai'
    chai.should()
    chai.use require 'chai-as-promised'

    Promise = require 'bluebird'
    path = require 'path'
    current_dir = path.dirname __filename
    process.chdir current_dir
    pkg = require '../package.json'
    seconds = 1000

    fs = Promise.promisifyAll require 'fs'

    logger = require 'winston'
    logger.remove logger.transports.Console
    logger.add logger.transports.Console, timestamp:on
    exec = (require 'exec-as-promised') logger

    describe 'Live test', ->
      docker =
        image: "shimaore/couchdb"
        container: "#{pkg.name}-test-container"
        name: "#{pkg.name}-test-instance"

      config =
        provisioning: 'http://127.0.0.1:5984/provisioning'
        provisioning_admin: 'http://127.0.0.1:5984/provisioning'
        interval: 3*seconds # keep it short so that we don't have to wait forever for the test suite to finish!!
        host: 'phone.example.net'

      before ->
        @timeout 90*seconds
        delay = 2
        Promise.resolve()
        .then -> exec "docker rm #{docker.container}"
        .catch -> true
        .then -> exec "docker run -d --net=host --name #{docker.container} #{docker.image}"

Write a new config.json file for the tests.

        .then ->
          fs.writeFileAsync '../config.json', JSON.stringify config

        .then ->
          logger.info "#{pkg.name} live tester: Waiting #{delay} seconds for image to be ready."
        .delay delay*seconds
        .then ->
          require 'superagent-as-promised'
          .get 'http://127.0.0.1:5984'
          .accept 'json'
        .then ({body}) ->
          logger.info "CouchDB #{body.version} is ready."


      after ->
        Promise.delay 500
        .then ->
          exec "docker kill #{docker.container}"

      run = require '../README'

      restart = (done) ->
        start = new Date()
        (_,cancel) ->
          stop = new Date()
          chai.expect(stop-start).to.be.gt config.interval
          Promise.delay(500).then ->
            cancel()
            done()
          Promise.resolve()

      it 'should report on creation', (done) ->
        @timeout 4*seconds
        run restart done
        .then ({db}) ->
          db.put
            _id:'number:33987654321'
            type:'number'
            number:'33987654321'
            account:'test'
            registrant_username:'foo'
            registrant_password:'bar'
            registrant_host: config.host

      it 'should report on update', (done) ->
        @timeout 4*seconds
        run restart done
        .then ({db}) ->
          db.get 'number:33987654321'
          .then (doc) ->
            doc.registrant_password = 'boo'
            db.put doc

      it 'should not report on put with not changes', (done) ->
        @timeout 5*seconds
        restarted = false
        _cancel = null
        Promise.delay(config.interval+500).then ->
          chai.expect(restarted).to.be.false
          _cancel?()
          done()
        run restart done
        .then ({db,cancel}) ->
          _cancel = cancel
          db.get 'number:33987654321'
          .then (doc) ->
            doc.registrant_password = 'boo'
            db.put doc

      it 'should report on deletion', (done) ->
        @timeout 5*seconds
        run restart done
        .then ({db}) ->
          db.get 'number:33987654321'
          .then (doc) ->
            doc._deleted = true
            db.put doc

      it 'should report on creation after deletion', (done) ->
        @timeout 5*seconds
        run restart done
        .then ({db}) ->
          db.put
            _id:'number:33987654321'
            type:'number'
            number:'33987654321'
            account:'test2'
            registrant_username:'test2'
            registrant_password:'foobar'
            registrant_host: config.host

      it 'should report on update after deletion', (done) ->
        @timeout 5*seconds
        run restart done
        .then ({db}) ->
          db.get 'number:33987654321'
          .then (doc) ->
            doc.registrant_password = 'barfoo'
            db.put doc

      it 'should report on deletion after deletion', (done) ->
        @timeout 5*seconds
        run restart done
        .then ({db}) ->
          db.get 'number:33987654321'
          .then (doc) ->
            doc._deleted = true
            db.put doc

      it 'should write a proper config file', ->
        fs.readFileAsync '../config.json'
        .then (buf) ->
          data = JSON.parse buf
          data.should.have.property 'provisioning', config.provisioning
          data.should.have.property 'provisioning_admin', config.provisioning_admin
          data.should.have.property 'host', config.host
          data.should.have.property 'update_seq'
