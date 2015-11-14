Restart the registrant
======================

Note: since there is not explicit way to restart an OpenSIPS instance, let's kill it and hope the OS monitoring tools will restart it.

    CWD = '/opt/ccnq3/src/applications/registrant'

    module.exports = (config,cancel) ->
      debug "Stopping registrant"
      exec 'npm stop', cwd:CWD
      .catch (error) ->
        debug "Stopping registrant failed (ignored)"
        null
      .then ->
        debug "Starting registrant"
        exec 'npm start', cwd:CWD, timeout:30*1000
      .catch ->
        debug "Starting registrant (again)"
        exec 'npm start', cwd:CWD

    Promise = require 'bluebird'
    exec = (require 'exec-as-promised')()
    pkg = require './package'
    debug = (require 'debug') "#{pkg.name}:restart-registrant"
