    Promise = require 'bluebird'
    dgram = Promise.promisifyAll require 'dgram'

    describe 'Restart', ->

      it 'should send UDP packet', (done) ->
        config =
          mi_host: '127.0.0.1'
          mi_port: 30010

        dgram = require 'dgram'
        s = dgram.createSocket 'udp4'
        s.on 'message', (msg) ->
          t = msg.toString 'utf8'
          if t is ':kill:\n'
            done()
          else
            done new Error "Received incorrect message #{t}"
          s.close()

        s.bindAsync config.mi_port
        .then ->
          restart = require '../restart-registrant'
          restart config
