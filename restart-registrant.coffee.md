Restart the registrant
======================

Note: since there is not explicit way to restart an OpenSIPS instance, let's kill it and hope the OS monitoring tools will restart it.

    CWD = '/opt/ccnq3/src/applications/registrant'

    module.exports = (config,cancel) ->
      exec 'npm stop', cwd:CWD
      .catch (error) ->
        logger.error "Stopping registrant failed: #{error}"
        null
      .then (stdout,stderr) ->
        logger.info "Stopping registrant", {stderr,stdout}
        exec 'npm start', cwd:CWD
      .catch (error) ->
        logger.error "Starting registrant failed: #{error}"
        null
      .then (stdout,stderr) ->
        logger.info "Starting registrant", {stderr,stdout}

    Promise = require 'bluebird'
    exec = Promise.promisify (require 'child_process').exec
    logger = require 'winston'
