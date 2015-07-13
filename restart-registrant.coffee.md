Restart the registrant
======================

Note: since there is not explicit way to restart an OpenSIPS instance, let's kill it and hope the OS monitoring tools will restart it.

    CWD = '/opt/ccnq3/src/applications/registrant'

    module.exports = (config,cancel) ->
      logger.info "Stopping registrant"
      exec 'npm stop', cwd:CWD
      .catch (error) ->
        logger.error "Stopping registrant failed (ignored)"
        null
      .then ->
        logger.info "Starting registrant"
        exec 'npm start', cwd:CWD
      .catch ->
        logger.info "Starting registrant (again)"
        exec 'npm start', cwd:CWD

    Promise = require 'bluebird'
    logger = require 'winston'
    exec = (require 'exec-as-promised') logger
