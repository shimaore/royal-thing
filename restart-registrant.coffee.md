Restart the registrant
======================

Note: since there is not explicit way to restart an OpenSIPS instance, let's kill it and hope the OS monitoring tools will restart it.

    module.exports = (config,cancel) ->
      mi_udp config.mi_host, config.mi_port, command 'kill'
      .catch -> true

    Promise = require 'bluebird'
    {mi_udp,command,parse} = require 'opensips/mi'
